---
lab:
  title: "Évaluer des grands modèles de langage à l’aide d’Azure\_Databricks et d’Azure\_OpenAI"
---

# Évaluer des grands modèles de langage à l’aide d’Azure Databricks et d’Azure OpenAI

L’évaluation d’un grand modèle de langage (LLM) nécessite une série d’étapes pour garantir que les performances du modèle répondent aux normes requises. MLflow LLM Evaluate, une fonctionnalité dans Azure Databricks, fournit une approche structurée de ce processus, comprenant la configuration de l’environnement, la définition des métriques d’évaluation et l’analyse des résultats. Cette évaluation est cruciale car les LLM ne peuvent pas être comparés facilement, ce qui rend les méthodes d’évaluation traditionnelles inadaptées.

Ce labo prend environ **20** minutes.

> **Remarque** : l’interface utilisateur d’Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner une ressource Azure OpenAI

Si vous n’en avez pas déjà une, approvisionnez une ressource Azure OpenAI dans votre abonnement Azure.

1. Connectez-vous au **portail Azure** à l’adresse `https://portal.azure.com`.
2. Créez une ressource **Azure OpenAI** avec les paramètres suivants :
    - **Abonnement** : *Sélectionner un abonnement Azure approuvé pour l’accès à Azure OpenAI Service*
    - **Groupe de ressources** : *sélectionnez ou créez un groupe de ressources*.
    - **Région** : *Choisir de manière **aléatoire** une région parmi les suivantes*\*
        - USA Est 2
        - Centre-Nord des États-Unis
        - Suède Centre
        - Suisse Ouest
    - **Nom** : *un nom unique de votre choix*
    - **Niveau tarifaire** : Standard S0

> \* Les ressources Azure OpenAI sont limitées par des quotas régionaux. Les régions répertoriées incluent le quota par défaut pour les types de modèle utilisés dans cet exercice. Le choix aléatoire d’une région réduit le risque d’atteindre sa limite de quota dans les scénarios où vous partagez un abonnement avec d’autres utilisateurs. Si une limite de quota est atteinte plus tard dans l’exercice, vous devrez peut-être créer une autre ressource dans une autre région.

3. Attendez la fin du déploiement. Accédez ensuite à la ressource Azure OpenAI déployée dans le portail Azure.

4. Dans le volet de gauche, sous **Gestion des ressources**, sélectionnez **Clés et points de terminaison**.

5. Copiez le point de terminaison et l’une des clés disponibles, car vous l’utiliserez plus loin dans cet exercice.

## Déployer le modèle nécessaire

Azure fournit un portail web appelé **Azure AI Studio**, que vous pouvez utiliser pour déployer, gérer et explorer des modèles. Vous allez commencer votre exploration d’Azure OpenAI en utilisant Azure AI Studio pour déployer un modèle.

> **Remarque** : lorsque vous utilisez Azure AI Studio, des boîtes de message qui suggèrent des tâches à effectuer peuvent être affichées. Vous pouvez les fermer et suivre les étapes de cet exercice.

1. Dans le portail Azure, sur la page **Vue d’ensemble** de votre ressource Azure OpenAI, faites défiler jusqu’à la section **Démarrer** et sélectionnez le bouton permettant d’accéder à **AI Studio**.
   
1. Dans Azure AI Studio, dans le panneau de gauche, sélectionnez la page **Deployments** et affichez vos modèles de déploiement existants. Si vous n’en avez pas encore, créez un déploiement du modèle **gpt-35-turbo** avec les paramètres suivants :
    - **Nom du déploiement** : *gpt-35-turbo*
    - **Modèle** : gpt-35-turbo
    - **Version du modèle** : par défaut
    - **Type de déploiement** : Standard
    - **Limite de débit de jetons par minute** : 5 000\*
    - **Filtre de contenu** : valeur par défaut
    - **Enable dynamic quota** : désactivé
    
> \* Une limite de débit de 5 000 jetons par minute est plus que suffisante pour effectuer cet exercice tout permettant à d’autres personnes d’utiliser le même abonnement.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

1. Connectez-vous au **portail Azure** à l’adresse `https://portal.azure.com`.
2. Créez une ressource **Azure Databricks** avec les paramètres suivants :
    - **Abonnement** : *sélectionnez le même abonnement Azure que celui utilisé pour créer votre ressource Azure OpenAI*
    - **Groupe de ressources** : *le groupe de ressources où vous avez créé votre ressource Azure OpenAI*
    - **Région** : *région dans laquelle vous avez créé votre ressource Azure OpenAI*
    - **Nom** : *un nom unique de votre choix*
    - **Niveau tarifaire** : *Premium* ou *Évaluation*

3. Sélectionnez **Examiner et créer**, puis attendez la fin du déploiement. Accédez ensuite à la ressource et lancez l’espace de travail.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version 13.3 LTS **<u>ML</u>** ou ultérieure du runtime dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour effectuer cet exercice et ignorer cette procédure.

1. Dans le Portail Azure, accédez au groupe de ressources où l’espace de travail Azure Databricks a été créé.
2. Sélectionnez votre ressource Azure Databricks Service.
3. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

> **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

4. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
5. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
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

6. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région.

## Installer les bibliothèques nécessaires

1. Sur la page de votre cluster, sélectionnez l’onglet **Bibliothèques**.

2. Sélectionnez **Installer**.

3. Sélectionnez **PyPI** comme bibliothèque source et installez `openai==1.42.0`.

## Créer une nouvelle instance Notebook

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
   
1. Donnez un nom à votre notebook et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

2. Dans la première cellule du notebook, exécutez le code suivant avec les informations d’accès que vous avez copiées au début de cet exercice afin d’affecter des variables d’environnement persistantes pour l’authentification lors de l’utilisation de ressources Azure OpenAI :

     ```python
    import os

    os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
    os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
    os.environ["AZURE_OPENAI_API_VERSION"] = "2023-03-15-preview"
     ```

## Évaluer un LLM avec une fonction personnalisée

Dans MLflow 2.8.0 et version ultérieure, `mlflow.evaluate()` prend en charge l’évaluation d’une fonction Python sans exiger la journalisation du modèle dans MLflow. Le processus implique de spécifier le modèle à évaluer, les métriques à calculer et les données d’évaluation, généralement un DataFrame Pandas. 

1. Exécutez le code suivant dans une nouvelle cellule pour définir un exemple de dataframe d’évaluation :

     ```python
    import pandas as pd

    eval_data = pd.DataFrame(
        {
            "inputs": [
                "What is MLflow?",
                "What is Spark?",
            ],
            "ground_truth": [
                "MLflow is an open-source platform for managing the end-to-end machine learning (ML) lifecycle. It was developed by Databricks, a company that specializes in big data and machine learning solutions. MLflow is designed to address the challenges that data scientists and machine learning engineers face when developing, training, and deploying machine learning models.",
                "Apache Spark is an open-source, distributed computing system designed for big data processing and analytics. It was developed in response to limitations of the Hadoop MapReduce computing model, offering improvements in speed and ease of use. Spark provides libraries for various tasks such as data ingestion, processing, and analysis through its components like Spark SQL for structured data, Spark Streaming for real-time data processing, and MLlib for machine learning tasks",
            ],
        }
    )
     ```

1. Exécutez le code suivant dans une nouvelle cellule pour initialiser un client pour votre ressource Azure OpenAI et définir votre fonction personnalisée :

     ```python
    import os
    import pandas as pd
    from openai import AzureOpenAI

    client = AzureOpenAI(
        azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key = os.getenv("AZURE_OPENAI_API_KEY"),
        api_version = os.getenv("AZURE_OPENAI_API_VERSION")
    )

    def openai_qa(inputs):
        answers = []
        system_prompt = "Please answer the following question in formal language."
        for index, row in inputs.iterrows():
            completion = client.chat.completions.create(
                model="gpt-35-turbo",
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": "{row}"},
                ],
            )
            answers.append(completion.choices[0].message.content)

        return answers

     ```

1. Exécutez le code suivant dans une nouvelle cellule pour créer une expérience et évaluer la fonction personnalisée avec les données d’évaluation :

     ```python
    import mlflow

    with mlflow.start_run() as run:
        results = mlflow.evaluate(
            openai_qa,
            eval_data,
            model_type="question-answering",
        )
     ```
Une fois l’exécution réussie, un lien sera généré vers la page d’expérience, sur laquelle vous pouvez vérifier les métriques du modèle. Pour `model_type="question-answering"`, les métriques par défaut sont **toxicity**, **ari_grade_level** et **flesch_kincaid_grade_level**.

## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
