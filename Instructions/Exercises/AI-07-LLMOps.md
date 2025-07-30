---
lab:
  title: Mettre en œuvre LLMOps avec Azure Databricks
---

# Mettre en œuvre LLMOps avec Azure Databricks

Azure Databricks fournit une plateforme unifiée qui simplifie le cycle de vie de l’IA, de la préparation des données au service et à la supervision des modèles, en optimisant les performances et l’efficacité des systèmes de Machine Learning. Elle prend en charge le développement d’applications d’IA générative, tirant parti de fonctionnalités telles que Unity Catalog pour la gouvernance des données, MLflow pour le suivi des modèles et Service de modèles Mosaic AI pour le déploiement de LLM.

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

1. Dans l’espace de travail Databricks, accédez à la section **Espace de travail**.

2. Sélectionnez **Créer**, puis **Notebook**.

3. Donnez un nom à votre notebook et sélectionnez le langage `Python`.

4. Dans la première cellule de code, entrez et exécutez le code suivant pour installer la bibliothèque OpenAI :
   
     ```python
    %pip install openai
     ```

5. Une fois l’installation terminée, redémarrez le noyau dans une nouvelle cellule :

     ```python
    %restart_python
     ```

## Journaliser le LLM à l’aide de MLflow

Les fonctionnalités de suivi LLM de MLflow vous permettent de journaliser les paramètres, les métriques, les prédictions et les artefacts. Les paramètres incluent des paires clé-valeur détaillant les configurations d’entrée, tandis que les métriques fournissent des mesures quantitatives des performances. Les prédictions englobent les invites d’entrée et les réponses du modèle, stockées en tant qu’artefacts pour faciliter leur récupération. Cette journalisation structurée permet de conserver un enregistrement détaillé de chaque interaction, ce qui facilite l’analyse et l’optimisation des LLM.

1. Dans une nouvelle cellule, exécutez le code suivant avec les informations d’accès que vous avez copiées au début de cet exercice afin d’affecter des variables d’environnement persistantes pour l’authentification lors de l’utilisation de ressources Azure OpenAI :

     ```python
    import os

    os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
    os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
    os.environ["AZURE_OPENAI_API_VERSION"] = "2023-03-15-preview"
     ```
1. Dans une nouvelle cellule, exécutez le code suivant pour initialiser votre client Azure OpenAI :

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
       azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
       api_key = os.getenv("AZURE_OPENAI_API_KEY"),
       api_version = os.getenv("AZURE_OPENAI_API_VERSION")
    )
     ```

1. Dans une nouvelle cellule, exécutez le code suivant pour initialiser le suivi MLflow et journaliser le modèle :     

     ```python
    import mlflow
    from openai import AzureOpenAI

    system_prompt = "Assistant is a large language model trained by OpenAI."

    mlflow.openai.autolog()

    with mlflow.start_run():

        response = client.chat.completions.create(
            model="gpt-35-turbo",
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": "Tell me a joke about animals."},
            ],
        )

        print(response.choices[0].message.content)
        mlflow.log_param("completion_tokens", response.usage.completion_tokens)
    mlflow.end_run()
     ```

La cellule ci-dessus démarre une expérience dans votre espace de travail et conserve les traces de chaque itération d’achèvement de conversation, permettant ainsi le suivi des entrées, des sorties et des métadonnées de chaque exécution.

## Analyser le modèle

1. Dans la barre latérale de gauche, sélectionnez **Expériences** puis sélectionnez l’expérience associée au notebook que vous avez utilisé pour cet exercice. Sélectionnez la dernière exécution et vérifiez dans la page Vue d’ensemble qu’il existe un paramètre journalisé : `completion_tokens`. Par défaut, la commande `mlflow.openai.autolog()` journalise les traces de chaque exécution, mais vous pouvez également journaliser des paramètres supplémentaires avec `mlflow.log_param()`, afin d’analyser le modèle ultérieurement.

1. Sélectionnez l’onglet **Traces**, puis sélectionnez la dernière trace créée. Vérifiez que le paramètre `completion_tokens` fait partie de la sortie de la trace :

   ![Interface utilisateur de trace MLFlow](./images/trace-ui.png)  

Lors de votre analyse du modèle, vous pouvez comparer les traces de différentes exécutions pour détecter une possible dérive des données. Recherchez des modifications significatives dans les distributions de données d’entrée, les prédictions de modèle ou les métriques de performances au fil du temps. Vous pouvez utiliser des tests statistiques ou des outils de visualisation pour faciliter cette analyse.

## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
