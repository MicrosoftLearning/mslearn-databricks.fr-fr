---
lab:
  title: Génération augmentée de récupération à l’aide d’Azure Databricks
---

# Génération augmentée de récupération à l’aide d’Azure Databricks

La génération augmentée de récupération (RAG) est une approche de pointe de l’IA qui améliore les grands modèles de langage en intégrant des sources de connaissances externes. Azure Databricks fournit une plateforme robuste pour développer des applications RAG, ce qui permet de transformer des données non structurées dans un format adapté à la récupération et à la génération de réponses. Ce processus implique une série d’étapes comprenant notamment la compréhension de la requête de l’utilisateur, la récupération des données pertinentes et la génération d’une réponse à l’aide d’un modèle de langage. L’infrastructure fournie par Azure Databricks prend en charge l’itération rapide et le déploiement d’applications RAG, garantissant ainsi des réponses de haute qualité et spécifiques aux domaines pouvant inclure des informations à jour et des connaissances propriétaires.

Ce labo prend environ **40** minutes.

> **Remarque** : l’interface utilisateur d’Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.
2. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***PowerShell***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Remarque** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *Bash*, basculez-le vers ***PowerShell***.

3. Notez que vous pouvez redimensionner Cloud Shell en faisant glisser la barre de séparation en haut du volet. Vous pouvez aussi utiliser les icônes **&#8212;**, **&#10530;** et **X** situées en haut à droite du volet pour réduire, agrandir et fermer ce dernier. Pour plus d’informations sur l’utilisation d’Azure Cloud Shell, consultez la [documentation Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. Dans le volet PowerShell, entrez les commandes suivantes pour cloner ce référentiel :

    ```powershell
   rm -r mslearn-databricks -f
   git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Une fois le référentiel cloné, entrez la commande suivante pour exécuter le script **setup.ps1**, qui approvisionne un espace de travail Azure Databricks dans une région disponible :

    ```powershell
   ./mslearn-databricks/setup.ps1
    ```

6. Si vous y êtes invité, choisissez l’abonnement à utiliser (uniquement si vous avez accès à plusieurs abonnements Azure).

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version runtime 15.4 LTS ou supérieure de **<u>ML</u>** dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour cet exercice et ignorer cette procédure.

1. Dans le portail Microsoft Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou le groupe de ressources contenant votre espace de travail Azure Databricks existant)
1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Machine Learning** : Activé
    - **Runtime Databricks** : 15.4 LTS
    - **Utiliser l’accélération photon** : <u>Non</u> sélectionné
    - **Type de collaborateur** : Standard_D4ds_v5
    - **Nœud simple** : Coché

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Installer les bibliothèques nécessaires

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.
1. Dans la première cellule de code, entrez et exécutez le code suivant pour installer les bibliothèques nécessaires :
   
    ```python
   %pip install faiss-cpu
   dbutils.library.restartPython()
    ```
   
## Ingérer des données

1. Dans un nouvel onglet du navigateur, téléchargez le [ fichier d'exemple.](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml) qui seront utilisées comme données dans cet exercice : `https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/enwiki-latest-pages-articles.xml`
1. De retour dans l'onglet Espace de travail Databricks, avec votre notebook ouvert, sélectionnez l’explorateur **Catalogue (CTRL + Alt + C)** et sélectionnez l'icône ➕ pour **Ajouter des données**.
1. Dans la page **Ajouter des données**, sélectionnez **Télécharger des fichiers vers DBFS**.
1. Dans la **page DBFS** , nommez le répertoire `RAG_lab` et chargez le fichier .xml que vous avez enregistré précédemment.
1. Dans la barre latérale, sélectionnez **Espace de travail** et ouvrez à nouveau votre bloc-notes.
1. Dans une nouvelle cellule de code, entrez le code suivant pour créer un dataframe à partir des données brutes :

    ```python
   from pyspark.sql import SparkSession

   # Create a Spark session
   spark = SparkSession.builder \
       .appName("RAG-DataPrep") \
       .getOrCreate()

   # Read the XML file
   raw_df = spark.read.format("xml") \
       .option("rowTag", "page") \
       .load("/FileStore/tables/RAG_lab/enwiki_latest_pages_articles.xml")

   # Show the DataFrame
   raw_df.show(5)

   # Print the schema of the DataFrame
   raw_df.printSchema()
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.
1. Dans une nouvelle cellule, exécutez le code suivant pour nettoyer et prétraiter les données afin d'extraire les champs de texte pertinents :

    ```python
   from pyspark.sql.functions import col

   clean_df = raw_df.select(col("title"), col("revision.text._VALUE").alias("text"))
   clean_df = clean_df.na.drop()
   clean_df.show(5)
    ```

## Générer des incorporations et implémenter la recherche vectorielle

FAISS (Facebook AI Similarity Search) est une bibliothèque de base de données vectorielle open source développée par Meta AI, conçue pour la recherche efficace de similitudes et le regroupement de vecteurs denses. FAISS permet d'effectuer des recherches rapides et évolutives par voisinage le plus proche, et peut être intégré à des systèmes de recherche hybrides afin de combiner la similarité vectorielle avec les techniques traditionnelles basées sur les mots-clés, améliorant ainsi la pertinence des résultats de recherche.

1. Dans une nouvelle cellule, exécutez le code suivant pour charger le modèle pré-entraîné `all-MiniLM-L6-v2` et convertir le texte en encodages :

    ```python
   from sentence_transformers import SentenceTransformer
   import numpy as np
    
   # Load pre-trained model
   model = SentenceTransformer('all-MiniLM-L6-v2')
    
   # Function to convert text to embeddings
   def text_to_embedding(text):
       embeddings = model.encode([text])
       return embeddings[0]
    
   # Convert the DataFrame to a Pandas DataFrame
   pandas_df = clean_df.toPandas()
    
   # Apply the function to get embeddings
   pandas_df['embedding'] = pandas_df['text'].apply(text_to_embedding)
   embeddings = np.vstack(pandas_df['embedding'].values)
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour créer et interroger l'index FAISS :

    ```python
   import faiss
    
   # Create a FAISS index
   d = embeddings.shape[1]  # dimension
   index = faiss.IndexFlatL2(d)  # L2 distance
   index.add(embeddings)  # add vectors to the index
    
   # Perform a search
   query_embedding = text_to_embedding("Anthropology fields")
   k = 1  # number of nearest neighbors
   distances, indices = index.search(np.array([query_embedding]), k)
    
   # Get the results
   results = pandas_df.iloc[indices[0]]
   display(results)
    ```

Vérifiez que la sortie recherche la page Wiki correspondant à l’invite de requête.

## Augmenter les requêtes avec des données récupérées

Nous pouvons désormais améliorer les fonctionnalités des grands modèles de langage en leur fournissant du contexte supplémentaire à partir de sources de données externes. Ainsi, les modèles peuvent générer des réponses plus précises et contextuellement plus pertinentes.

1. Dans une nouvelle cellule, exécutez le code suivant pour combiner les données récupérées avec la requête de l’utilisateur afin de créer une invite enrichie pour le LLM.

    ```python
   from transformers import pipeline
    
   # Load the summarization model
   summarizer = pipeline("summarization", model="facebook/bart-large-cnn", framework="pt")
    
   # Extract the string values from the DataFrame column
   text_data = results["text"].tolist()
    
   # Pass the extracted text data to the summarizer function
   summary = summarizer(text_data, max_length=512, min_length=100, do_sample=True)
    
   def augment_prompt(query_text):
       context = " ".join([item['summary_text'] for item in summary])
       return f"{context}\n\nQuestion: {query_text}\nAnswer:"
    
   prompt = augment_prompt("Explain the significance of Anthropology")
   print(prompt)
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour utiliser un LLM pour générer des réponses.

    ```python
   from transformers import GPT2LMHeadModel, GPT2Tokenizer

   tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
   model = GPT2LMHeadModel.from_pretrained("gpt2")

   inputs = tokenizer(prompt, return_tensors="pt")
   outputs = model.generate(
       inputs["input_ids"], 
       max_length=300, 
       num_return_sequences=1, 
       repetition_penalty=2.0, 
       top_k=50, 
       top_p=0.95, 
       temperature=0.7,
       do_sample=True
   )
   response = tokenizer.decode(outputs[0], skip_special_tokens=True)

   print(response)
    ```

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
