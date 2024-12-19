---
lab:
  title: Explorer des grands modèles de langage avec Azure Databricks
---

# Explorer des grands modèles de langage avec Azure Databricks

Les grands modèles de langage (LLMs) peuvent être un atout puissant pour les tâches de traitement du langage naturel (NLP) lorsqu’elles sont intégrées à Azure Databricks et aux transformateurs Hugging Face. Azure Databricks fournit une plateforme transparente pour accéder, ajuster et déployer des LLM, y compris des modèles préentraînés issus de la bibliothèque complète de Hugging Face. Pour l’inférence de modèle, la classe de pipelines de Hugging Face simplifie l’utilisation de modèles préentraînés en prenant en charge un large éventail de tâches NLP directement dans l’environnement Databricks.

Ce labo prend environ **30** minutes.

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

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Présentation de Delta Lake](https://docs.microsoft.com/azure/databricks/delta/delta-intro) dans la documentation Azure Databricks.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version 13.3 LTS **<u>ML</u>** ou ultérieure du runtime dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour effectuer cet exercice et ignorer cette procédure.

1. Dans le portail Microsoft Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou le groupe de ressources contenant votre espace de travail Azure Databricks existant)
1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Mode cluster** : nœud unique
    - **Mode d’accès** : un seul utilisateur (*avec votre compte d’utilisateur sélectionné*)
    - **Version du runtime Databricks** : *Sélectionnez l’édition **<u>ML</u>** de la dernière version non bêta du runtime (**Not** version du runtime standard) qui :*
        - *N’utilise **pas** de GPU*
        - *Inclut Scala > **2.11***
        - *Inclut Spark > **3.4***
    - **Utiliser l’accélération photon** : <u>Non</u> sélectionné
    - **Type de nœud** : Standard_D4ds_v5
    - **Arrêter après** *20* **minutes d’inactivité**

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Installer les bibliothèques nécessaires

1. Sur la page de votre cluster, sélectionnez l’onglet **Bibliothèques**.

2. Sélectionnez **Installer**.

3. Sélectionnez **PyPI** comme bibliothèque source et saisissez `transformers==4.44.0` dans le champ **Package**.

4. Sélectionnez **Installer**.

## Charger des modèles préentraînés

1. Dans l’espace de travail Databricks, accédez à la section **Espace de travail**.

2. Sélectionnez **Créer**, puis **Notebook**.

3. Donnez un nom à votre notebook et sélectionnez le langage `Python`.

4. Saisissez le code suivant dans la première cellule de code et exécutez-le :

     ```python
    from transformers import pipeline

    # Load the summarization model
    summarizer = pipeline("summarization")

    # Load the sentiment analysis model
    sentiment_analyzer = pipeline("sentiment-analysis")

    # Load the translation model
    translator = pipeline("translation_en_to_fr")

    # Load a general purpose model for zero-shot classification and few-shot learning
    classifier = pipeline("zero-shot-classification")
     ```
Ce code charge tous les modèles nécessaires pour les tâches NLP présentées dans cet exercice.

### Résumer le texte

Un pipeline de résumé génère des résumés concis à partir de longs textes. Il est possible de déterminer le niveau de précision ou de créativité du résumé généré en spécifiant une plage de longueur (`min_length`, `max_length`) et si celui-ci utilise l’échantillonnage ou non (`do_sample`). 

1. Dans une nouvelle cellule de code, saisissez le code suivant :

     ```python
    text = "Large language models (LLMs) are advanced AI systems capable of understanding and generating human-like text by learning from vast datasets. These models, which include OpenAI's GPT series and Google's BERT, have transformed the field of natural language processing (NLP). They are designed to perform a wide range of tasks, from translation and summarization to question-answering and creative writing. The development of LLMs has been a significant milestone in AI, enabling machines to handle complex language tasks with increasing sophistication. As they evolve, LLMs continue to push the boundaries of what's possible in machine learning and artificial intelligence, offering exciting prospects for the future of technology."
    summary = summarizer(text, max_length=75, min_length=25, do_sample=False)
    print(summary)
     ```

2. Exécutez la cellule pour afficher le texte résumé.

### Analyser des sentiments

Le pipeline d’analyse des sentiments détermine le sentiment d’un texte donné. Il classe le texte en catégories telles que positif, négatif ou neutre.

1. Dans une nouvelle cellule de code, saisissez le code suivant :

     ```python
    text = "I love using Azure Databricks for NLP tasks!"
    sentiment = sentiment_analyzer(text)
    print(sentiment)
     ```

2. Exécutez la cellule pour afficher le résultat de l’analyse des sentiments.

### Traduire du texte

Le pipeline de traduction convertit le texte d’une langue à une autre. Dans cet exercice, la tâche utilisée était `translation_en_to_fr`, ce qui signifie qu’elle traduit tout texte de l’anglais au français.

1. Dans une nouvelle cellule de code, saisissez le code suivant :

     ```python
    text = "Hello, how are you?"
    translation = translator(text)
    print(translation)
     ```

2. Exécutez la cellule pour afficher le texte traduit en français.

### Classifier le texte

Le pipeline de classification « zero shot » permet à un modèle de classer du texte dans des catégories qu’il n’a pas rencontrées au cours de son entraînement. Par conséquent, elle nécessite des libellés prédéfinis en tant que paramètre `candidate_labels`.

1. Dans une nouvelle cellule de code, saisissez le code suivant :

     ```python
    text = "Azure Databricks is a powerful platform for big data analytics."
    labels = ["technology", "health", "finance"]
    classification = classifier(text, candidate_labels=labels)
    print(classification)
     ```

2. Exécutez la cellule pour afficher les résultats de la classification « zero shot ».

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
