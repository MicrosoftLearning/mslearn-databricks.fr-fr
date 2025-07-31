---
lab:
  title: "Raisonnement en plusieurs étapes avec LangChain à l’aide d’Azure\_Databricks et Azure\_OpenAI"
---

# Raisonnement en plusieurs étapes avec LangChain à l’aide d’Azure Databricks et Azure OpenAI

Le raisonnement à plusieurs étapes est une approche de pointe de l’IA qui implique de décomposer des problèmes complexes en étapes plus petites et mieux gérables. LangChain, une infrastructure logicielle, facilite la création d’applications qui tirent parti des grands modèles de langage (LLM). Lorsqu’il est intégré à Azure Databricks, LangChain permet un chargement transparent des données, un wrapping de modèle et le développement d’agents d’IA sophistiqués. Cette combinaison est particulièrement puissante pour gérer des tâches complexes qui nécessitent une compréhension approfondie du contexte et la capacité de raisonner en plusieurs étapes.

Ce labo prend environ **30** minutes.

> **Remarque** : l’interface utilisateur d’Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner une ressource Azure OpenAI

Si vous n’en avez pas déjà une, approvisionnez une ressource Azure OpenAI dans votre abonnement Azure.

1. Connectez-vous au **portail Azure** à l’adresse `https://portal.azure.com`.
1. Créez une ressource **Azure OpenAI** avec les paramètres suivants :
    - **Abonnement** : *Sélectionner un abonnement Azure approuvé pour l’accès à Azure OpenAI Service*
    - **Groupe de ressources** : *sélectionnez ou créez un groupe de ressources*.
    - **Région** : *Choisir de manière **aléatoire** une région parmi les suivantes*\*
        - Australie Est
        - Est du Canada
        - USA Est
        - USA Est 2
        - France Centre
        - Japon Est
        - Centre-Nord des États-Unis
        - Suède Centre
        - Suisse Nord
        - Sud du Royaume-Uni
    - **Nom** : *un nom unique de votre choix*
    - **Niveau tarifaire** : Standard S0

> \* Les ressources Azure OpenAI sont limitées par des quotas régionaux. Les régions répertoriées incluent le quota par défaut pour les types de modèle utilisés dans cet exercice. Le choix aléatoire d’une région réduit le risque d’atteindre sa limite de quota dans les scénarios où vous partagez un abonnement avec d’autres utilisateurs. Si une limite de quota est atteinte plus tard dans l’exercice, vous devrez peut-être créer une autre ressource dans une autre région.

1. Attendez la fin du déploiement. Accédez ensuite à la ressource Azure OpenAI déployée dans le portail Azure.

1. Dans le volet de gauche, sous **Gestion des ressources**, sélectionnez **Clés et points de terminaison**.

1. Copiez le point de terminaison et l’une des clés disponibles, car vous l’utiliserez plus loin dans cet exercice.

## Déployer les modèles nécessaires

Azure fournit un portail web appelé **Azure AI Foundry**, que vous pouvez utiliser pour déployer, gérer et explorer des modèles. Vous allez commencer votre exploration d’Azure OpenAI à l’aide d’Azure AI Foundry pour déployer un modèle.

> **Remarque** : Lorsque vous utilisez Azure AI Foundry, les boîtes de message qui suggèrent des tâches que vous devez effectuer peuvent s’afficher. Vous pouvez les fermer et suivre les étapes de cet exercice.

1. Dans le portail Azure, accédez à la page **Vue d’ensemble** de votre ressource Azure OpenAI, faites défiler jusqu’à la section **Démarrer**, puis sélectionnez le bouton pour accéder à **Azure AI Foundry**.
   
1. Dans Azure AI Foundry, sélectionnez la page **Deployments** dans le volet de gauche et affichez vos déploiements de modèles existants. Si vous n’en avez pas encore, créez un déploiement du modèle **gpt-4o** avec les paramètres suivants :
    - **Nom du déploiement** : *gpt-4o*
    - **Type de déploiement** : Standard
    - **Model version** : *utiliser la version par défaut*
    - **Limitation du débit en jetons par minute** : 10 000\*
    - **Filtre de contenu** : valeur par défaut
    - **Enable dynamic quota** : désactivé
    
1. Retournez à la page **Déploiements** et créez un déploiement du modèle **text-embedding-ada-002** avec les paramètres suivants :
    - **Nom du déploiement** : *text-embedding-ada-002*
    - **Type de déploiement** : Standard
    - **Model version** : *utiliser la version par défaut*
    - **Limitation du débit en jetons par minute** : 10 000\*
    - **Filtre de contenu** : valeur par défaut
    - **Enable dynamic quota** : désactivé

> \* Une limite de débit de 10 000 jetons par minute est plus que suffisante pour effectuer cet exercice tout en permettant à d’autres personnes d’utiliser le même abonnement.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

1. Connectez-vous au **portail Azure** à l’adresse `https://portal.azure.com`.
1. Créez une ressource **Azure Databricks** avec les paramètres suivants :
    - **Abonnement** : *sélectionnez le même abonnement Azure que celui utilisé pour créer votre ressource Azure OpenAI*
    - **Groupe de ressources** : *le groupe de ressources où vous avez créé votre ressource Azure OpenAI*
    - **Région** : *région dans laquelle vous avez créé votre ressource Azure OpenAI*
    - **Nom** : *un nom unique de votre choix*
    - **Niveau tarifaire** : *Premium* ou *Évaluation*

1. Sélectionnez **Examiner et créer**, puis attendez la fin du déploiement. Accédez ensuite à la ressource et lancez l’espace de travail.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d'un cluster avec une version d'exécution 16.4 LTS **<u>ML</u>** ou supérieure dans votre espace de travail Azure Databricks, vous pouvez l'utiliser pour réaliser cet exercice et ignorer cette procédure.

1. Dans le Portail Azure, accédez au groupe de ressources où l’espace de travail Azure Databricks a été créé.
1. Sélectionnez votre ressource Azure Databricks Service.
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

> **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Machine Learning** : Activé
    - **Runtime Databricks** : 16.4-LTS
    - **Utiliser l’accélération photon** : <u>Non</u> sélectionné
    - **Type de collaborateur** : Standard_D4ds_v5
    - **Nœud simple** : Coché

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région.

## Installer les bibliothèques nécessaires

1. Dans l’espace de travail Databricks, accédez à la section **Espace de travail**.
1. Sélectionnez **Créer**, puis **Notebook**.
1. Donnez un nom à votre notebook et sélectionnez le langage `Python`.
1. Dans la première cellule de code, entrez et exécutez le code suivant pour installer les bibliothèques nécessaires :
   
    ```python
   %pip install langchain openai langchain_openai faiss-cpu
    ```

1. Une fois l’installation terminée, redémarrez le noyau dans une nouvelle cellule :

    ```python
   %restart_python
    ```

1. Dans une nouvelle cellule, définissez les paramètres d’authentification qui seront utilisés pour initialiser les modèles OpenAI, en remplaçant `your_openai_endpoint` et `your_openai_api_key` par la clé et le point de terminaison copiés précédemment à partir de votre ressource OpenAI :

    ```python
   endpoint = "your_openai_endpoint"
   key = "your_openai_api_key"
    ```
    
## Créer un index vectoriel et stocker des intégrations

Un index vectoriel est une structure de données spécialisée qui permet un stockage et une récupération efficaces des données vectorielles à haute dimension, ce qui est essentiel pour effectuer des recherches rapides de similarité et des requêtes de type « plus proche voisin ». Les intégrations, d’autre part, sont des représentations numériques d’objets qui capturent leur signification dans une forme vectorielle, ce qui permet aux machines de traiter et de comprendre différents types de données, y compris du texte et des images.

1. Dans une nouvelle cellule, exécutez le code suivant pour charger l’échantillon du jeu de données :

    ```python
   from langchain_core.documents import Document

   documents = [
        Document(page_content="Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.", metadata={"date_created": "2024-08-22"}),
        Document(page_content="LangChain is a framework designed to simplify the creation of applications using large language models.", metadata={"date_created": "2024-08-22"}),
        Document(page_content="GPT-4 is a powerful language model developed by OpenAI.", metadata={"date_created": "2024-08-22"})
   ]
   ids = ["1", "2", "3"]
    ```
     
1. Dans une nouvelle cellule, exécutez le code suivant pour générer des intégrations à l’aide du modèle `text-embedding-ada-002` :

    ```python
   from langchain_openai import AzureOpenAIEmbeddings
     
   embedding_function = AzureOpenAIEmbeddings(
       deployment="text-embedding-ada-002",
       model="text-embedding-ada-002",
       azure_endpoint=endpoint,
       openai_api_key=key,
       chunk_size=1
   )
    ```
     
1. Dans une nouvelle cellule, exécutez le code suivant pour créer un index vectoriel à l’aide du premier exemple de texte comme référence pour la dimension vectorielle :

    ```python
   import faiss
      
   index = faiss.IndexFlatL2(len(embedding_function.embed_query("Azure Databricks is a fast, easy, and collaborative Apache Spark-based analytics platform.")))
    ```

## Créer une chaîne basée sur un récupérateur

Un composant de récupérateur récupère des données et documents pertinents en fonction d’une requête. Cela est particulièrement utile dans les applications qui nécessitent l’intégration de grandes quantités de données pour l’analyse, comme dans les systèmes de génération augmentée de récupération.

1. Dans une nouvelle cellule, exécutez le code suivant pour créer un récupérateur capable de rechercher l’index vectoriel pour les textes les plus similaires.

    ```python
   from langchain.vectorstores import FAISS
   from langchain_core.vectorstores import VectorStoreRetriever
   from langchain_community.docstore.in_memory import InMemoryDocstore

   vector_store = FAISS(
       embedding_function=embedding_function,
       index=index,
       docstore=InMemoryDocstore(),
       index_to_docstore_id={}
   )
   vector_store.add_documents(documents=documents, ids=ids)
   retriever = VectorStoreRetriever(vectorstore=vector_store)
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour créer un système AQ à l’aide du récupérateur et du modèle `gpt-4o` :
    
    ```python
   from langchain_openai import AzureChatOpenAI
   from langchain_core.prompts import ChatPromptTemplate
   from langchain.chains.combine_documents import create_stuff_documents_chain
   from langchain.chains import create_retrieval_chain
     
   llm = AzureChatOpenAI(
       deployment_name="gpt-4o",
       model_name="gpt-4o",
       azure_endpoint=endpoint,
       api_version="2023-03-15-preview",
       openai_api_key=key,
   )

   system_prompt = (
       "Use the given context to answer the question. "
       "If you don't know the answer, say you don't know. "
       "Use three sentences maximum and keep the answer concise. "
       "Context: {context}"
   )

   prompt1 = ChatPromptTemplate.from_messages([
       ("system", system_prompt),
       ("human", "{input}")
   ])

   chain = create_stuff_documents_chain(llm, prompt1)

   qa_chain1 = create_retrieval_chain(retriever, chain)
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour tester le système AQ :

    ```python
   result = qa_chain1.invoke({"input": "What is Azure Databricks?"})
   print(result)
    ```

    La sortie du résultat doit afficher une réponse basée sur le document pertinent présent dans l’échantillon du jeu de données, ainsi que le texte génératif produit par le LLM.

## Combiner des chaînes dans un système à plusieurs chaînes

Langchain est un outil polyvalent qui permet la combinaison de plusieurs chaînes dans un système à plusieurs chaînes, améliorant ainsi les fonctionnalités des modèles linguistiques. Ce processus implique la chaîne de différents composants qui peuvent traiter les entrées en parallèle ou en séquence, synthétisant ainsi une réponse finale.

1. Dans une nouvelle cellule, exécutez le code suivant pour créer une deuxième chaîne.

    ```python
   from langchain_core.prompts import ChatPromptTemplate
   from langchain_core.output_parsers import StrOutputParser

   prompt2 = ChatPromptTemplate.from_template("Create a social media post based on this summary: {summary}")

   qa_chain2 = ({"summary": qa_chain1} | prompt2 | llm | StrOutputParser())
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour appeler une chaîne à plusieurs étapes avec une entrée donnée :

    ```python
   result = qa_chain2.invoke({"input": "How can we use LangChain?"})
   print(result)
    ```

    La première chaîne fournit une réponse à l’entrée basée sur l’échantillon du jeu de données fourni, tandis que la deuxième chaîne crée un post de réseau social basé sur la sortie de la première chaîne. Cette approche vous permet de gérer des tâches de traitement de texte plus complexes en chaînant plusieurs étapes ensemble.

## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
