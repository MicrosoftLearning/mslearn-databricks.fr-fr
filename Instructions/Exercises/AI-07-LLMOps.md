---
lab:
  title: Implémentation de LLMOps avec Azure Databricks
---

# Implémentation de LLMOps avec Azure Databricks

Azure Databricks fournit une plateforme unifiée qui simplifie le cycle de vie de l’IA, de la préparation des données au service et à la supervision des modèles, en optimisant les performances et l’efficacité des systèmes de Machine Learning. Elle prend en charge le développement d’applications d’IA générative, tirant parti de fonctionnalités telles que Unity Catalog pour la gouvernance des données, MLflow pour le suivi des modèles et Service de modèles Mosaic AI pour le déploiement de LLM.

Ce labo prend environ **20** minutes.

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

4. Dans la première cellule de code, entrez et exécutez le code suivant pour installer les bibliothèques nécessaires :
   
     ```python
    %pip install azure-ai-openai flask
     ```

5. Une fois l’installation terminée, redémarrez le noyau dans une nouvelle cellule :

     ```python
    %restart_python
     ```

## Journaliser le LLM à l’aide de MLflow

1. Dans une nouvelle cellule, exécutez le code suivant pour initialiser votre client Azure OpenAI :

     ```python
    from azure.ai.openai import OpenAIClient

    client = OpenAIClient(api_key="<Your_API_Key>")
    model = client.get_model("gpt-3.5-turbo")
     ```

1. Dans une nouvelle cellule, exécutez le code suivant pour initialiser le suivi MLflow :     

     ```python
    import mlflow

    mlflow.set_tracking_uri("databricks")
    mlflow.start_run()
     ```

1. Dans une nouvelle cellule, exécutez le code suivant pour journaliser le modèle :

     ```python
    mlflow.pyfunc.log_model("model", python_model=model)
    mlflow.end_run()
     ```

## Déployer le modèle

1. Créez un notebook et, dans sa première cellule, exécutez le code suivant pour créer une API REST pour le modèle :

     ```python
    from flask import Flask, request, jsonify
    import mlflow.pyfunc

    app = Flask(__name__)

    @app.route('/predict', methods=['POST'])
    def predict():
        data = request.json
        model = mlflow.pyfunc.load_model("model")
        prediction = model.predict(data["input"])
        return jsonify(prediction)

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=5000)
     ```

## Analyser le modèle

1. Dans votre premier notebook, créez une cellule et exécutez le code suivant pour activer la journalisation automatique MLflow :

     ```python
    mlflow.autolog()
     ```

1. Dans une nouvelle cellule, exécutez le code suivant pour suivre les prédictions et les données d’entrée.

     ```python
    mlflow.log_param("input", data["input"])
    mlflow.log_metric("prediction", prediction)
     ```

1. Dans une nouvelle cellule, exécutez le code suivant pour surveiller la dérive des données :

     ```python
    import pandas as pd
    from evidently.dashboard import Dashboard
    from evidently.tabs import DataDriftTab

    report = Dashboard(tabs=[DataDriftTab()])
    report.calculate(reference_data=historical_data, current_data=current_data)
    report.show()
     ```

Une fois que vous avez commencé à surveiller le modèle, vous pouvez configurer des pipelines de réentraînement automatisés si des dérives de données sont détectées.

## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
