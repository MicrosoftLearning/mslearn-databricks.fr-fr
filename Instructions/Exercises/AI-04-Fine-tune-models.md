---
lab:
  title: "Ajustement des grands modèles de langage à l’aide d’Azure\_Databricks et d’Azure\_OpenAI"
---

# Ajustement des grands modèles de langage à l’aide d’Azure Databricks et d’Azure OpenAI

Avec Azure Databricks, les utilisateurs peuvent désormais tirer parti de la puissance des LLM pour des tâches spécialisées en les ajustant avec leurs propres données, afin d’améliorer les performances spécifiques au domaine. Pour ajuster un modèle de langage à l’aide d’Azure Databricks, vous pouvez utiliser l’interface d’entraînement du modèle Mosaic AI qui simplifie le processus de paramétrage complet du modèle. Cette fonctionnalité vous permet d’affiner un modèle avec vos données personnalisées, avec des points de contrôle enregistrés dans MLflow, afin que vous conserviez un contrôle total sur le modèle ainsi ajusté.

Ce labo prend environ **60** minutes.

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

6. Lancez Cloud Shell et exécutez `az account get-access-token` pour obtenir un jeton d’autorisation temporaire pour les tests d’API. Conservez-le, ainsi que le point de terminaison et la clé copiés précédemment.

## Déployer le modèle nécessaire

Azure fournit un portail web appelé **Azure AI Studio**, que vous pouvez utiliser pour déployer, gérer et explorer des modèles. Vous allez commencer votre exploration d’Azure OpenAI en utilisant Azure AI Studio pour déployer un modèle.

> **Remarque** : lorsque vous utilisez Azure AI Studio, des boîtes de message qui suggèrent des tâches à effectuer peuvent être affichées. Vous pouvez les fermer et suivre les étapes de cet exercice.

1. Dans le portail Azure, sur la page **Vue d’ensemble** de votre ressource Azure OpenAI, faites défiler jusqu’à la section **Démarrer** et sélectionnez le bouton permettant d’accéder à **AI Studio**.
   
1. Dans Azure AI Studio, dans le panneau de gauche, sélectionnez la page **Deployments** et affichez vos modèles de déploiement existants. Si vous n’en avez pas encore, créez un déploiement du modèle **gpt-35-turbo** avec les paramètres suivants :
    - **Nom du déploiement** : *gpt-35-turbo-0613*
    - **Modèle** : gpt-35-turbo
    - **Version du modèle** : 0613
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

3. Sélectionnez **PyPI** comme bibliothèque source et installez les packages Python suivants :
   - `numpy==2.1.0`
   - `requests==2.32.3`
   - `openai==1.42.0`
   - `tiktoken==0.7.0`

## Créer un notebook et ingérer des données

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
   
1. Donnez un nom à votre notebook et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

2. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/fine_tuning
    mkdir /dbfs/fine_tuning
    wget -O /dbfs/fine_tuning/training_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/training_set.jsonl
    wget -O /dbfs/fine_tuning/validation_set.jsonl https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/validation_set.jsonl
     ```

3. Dans une nouvelle cellule, exécutez le code suivant avec les informations d’accès que vous avez copiées au début de cet exercice afin d’affecter des variables d’environnement persistantes pour l’authentification lors de l’utilisation de ressources Azure OpenAI :

     ```python
    import os

    os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
    os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
    os.environ["TEMP_AUTH_TOKEN"] = "your_access_token"
     ```
     
## Nombre de jetons validés

`training_set.jsonl` et `validation_set.jsonl` sont constitués d’exemples de conversation différents entre `user` et `assistant`, qui serviront de points de données pour l’entraînement et la validation du modèle ajusté. Les exemples individuels doivent rester sous la limite des jetons d’entrée du modèle `gpt-35-turbo` de 4 096 jetons.

1. Dans une nouvelle cellule, exécutez le code suivant pour valider le nombre de jetons pour chaque fichier :

   ```python
    import json
    import tiktoken
    import numpy as np
    from collections import defaultdict

    encoding = tiktoken.get_encoding("cl100k_base")

    def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
        num_tokens = 0
        for message in messages:
            num_tokens += tokens_per_message
            for key, value in message.items():
                num_tokens += len(encoding.encode(value))
                if key == "name":
                    num_tokens += tokens_per_name
        num_tokens += 3
        return num_tokens

    def num_assistant_tokens_from_messages(messages):
        num_tokens = 0
        for message in messages:
            if message["role"] == "assistant":
                num_tokens += len(encoding.encode(message["content"]))
        return num_tokens

    def print_distribution(values, name):
        print(f"\n##### Distribution of {name}:")
        print(f"min / max: {min(values)}, {max(values)}")
        print(f"mean / median: {np.mean(values)}, {np.median(values)}")

    files = ['/dbfs/fine_tuning/training_set.jsonl', '/dbfs/fine_tuning/validation_set.jsonl']

    for file in files:
        print(f"File: {file}")
        with open(file, 'r', encoding='utf-8') as f:
            dataset = [json.loads(line) for line in f]

        total_tokens = []
        assistant_tokens = []

        for ex in dataset:
            messages = ex.get("messages", {})
            total_tokens.append(num_tokens_from_messages(messages))
            assistant_tokens.append(num_assistant_tokens_from_messages(messages))

        print_distribution(total_tokens, "total tokens")
        print_distribution(assistant_tokens, "assistant tokens")
        print('*' * 75)
   ```

## Charger des fichiers d’ajustement dans Azure OpenAI

Avant de commencer à ajuster le modèle, vous devez initialiser un client OpenAI et ajouter les fichiers d’ajustement à son environnement, en générant des ID de fichiers qui seront utilisés pour initialiser la tâche.

1. Exécutez le code suivant dans une nouvelle cellule :

     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-05-01-preview"  # This API version or later is required to access seed/events/checkpoint features
    )

    training_file_name = '/dbfs/fine_tuning/training_set.jsonl'
    validation_file_name = '/dbfs/fine_tuning/validation_set.jsonl'

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
     ```

## Soumettre une tâche d’ajustement

Maintenant que les fichiers d’ajustement ont été correctement chargés, vous pouvez soumettre votre tâche d’entraînement à l’ajustement : Il n’est pas rare que l’entraînement prenne plus d’une heure. Une fois l’entraînement terminé, vous pouvez voir les résultats dans Azure AI Studio en sélectionnant l’option **Ajustement** dans le volet gauche.

1. Exécutez le code suivant dans une nouvelle cellule pour démarrer la tâche d’entraînement à l’ajustement :

     ```python
    response = client.fine_tuning.jobs.create(
        training_file = training_file_id,
        validation_file = validation_file_id,
        model = "gpt-35-turbo-0613",
        seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
    )

    job_id = response.id
     ```

Le paramètre `seed` contrôle la reproductibilité de la tâche d’ajustement. La transmission des mêmes paramètres de travail et seed doit produire les mêmes résultats, mais peut différer dans de rares cas. Si aucune seed n’est spécifiée, une seed est générée automatiquement.

2. Vous pouvez exécuter le code suivant dans une nouvelle cellule pour surveiller l’état de la tâche d’ajustement :

     ```python
    print("Job ID:", response.id)
    print("Status:", response.status)
     ```

3. Lorsque l’état de la tâche passe à `succeeded`, exécutez le code suivant pour obtenir les résultats finaux :

     ```python
    response = client.fine_tuning.jobs.retrieve(job_id)

    print(response.model_dump_json(indent=2))
    fine_tuned_model = response.fine_tuned_model
     ```
   
## Déployer un modèle ajusté

Une fois en possession de votre modèle ajusté, vous pouvez le déployer en tant que modèle personnalisé et l’utiliser comme n’importe quel autre modèle déployé dans le **terrain de jeu de conversation d’Azure OpenAI Studio** ou via l’API de saisie semi-automatique de conversation.

1. Exécutez le code suivant dans une nouvelle cellule pour déployer votre modèle ajusté :
   
     ```python
    import json
    import requests

    token = os.getenv("TEMP_AUTH_TOKEN")
    subscription = "<YOUR_SUBSCRIPTION_ID>"
    resource_group = "<YOUR_RESOURCE_GROUP_NAME>"
    resource_name = "<YOUR_AZURE_OPENAI_RESOURCE_NAME>"
    model_deployment_name = "gpt-35-turbo-ft"

    deploy_params = {'api-version': "2023-05-01"}
    deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

    deploy_data = {
        "sku": {"name": "standard", "capacity": 1},
        "properties": {
            "model": {
                "format": "OpenAI",
                "name": "<YOUR_FINE_TUNED_MODEL>",
                "version": "1"
            }
        }
    }
    deploy_data = json.dumps(deploy_data)

    request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

    print('Creating a new deployment...')

    r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

    print(r)
    print(r.reason)
    print(r.json())
     ```

2. Exécutez le code suivant dans une nouvelle cellule pour utiliser votre modèle personnalisé dans un appel avec saisie semi-automatique de conversation :
   
     ```python
    import os
    from openai import AzureOpenAI

    client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = "2024-02-01"
    )

    response = client.chat.completions.create(
        model = "gpt-35-turbo-ft", # model = "Custom deployment name you chose for your fine-tuning model"
        messages = [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"},
            {"role": "assistant", "content": "Yes, customer managed keys are supported by Azure OpenAI."},
            {"role": "user", "content": "Do other Azure AI services support this too?"}
        ]
    )

    print(response.choices[0].message.content)
     ```
 
## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
