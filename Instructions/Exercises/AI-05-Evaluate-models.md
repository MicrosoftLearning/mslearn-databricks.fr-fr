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
    
> \* Une limite de débit de 10 000 jetons par minute est plus que suffisante pour effectuer cet exercice tout en permettant à d’autres personnes d’utiliser le même abonnement.

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

> **Conseil** : Si vous disposez déjà d'un cluster avec une version d'exécution 16.4 LTS **<u>ML</u>** ou supérieure dans votre espace de travail Azure Databricks, vous pouvez l'utiliser pour réaliser cet exercice et ignorer cette procédure.

1. Dans le Portail Azure, accédez au groupe de ressources où l’espace de travail Azure Databricks a été créé.
2. Sélectionnez votre ressource Azure Databricks Service.
3. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

> **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

4. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
5. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Machine Learning** : Activé
    - **Runtime Databricks** : 16.4-LTS
    - **Utiliser l’accélération photon** : <u>Non</u> sélectionné
    - **Type de collaborateur** : Standard_D4ds_v5
    - **Nœud simple** : Coché

6. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région.

## Installer les bibliothèques nécessaires

1. Dans l’espace de travail Databricks, accédez à la section **Espace de travail**.
1. Sélectionnez **Créer**, puis **Notebook**.
1. Donnez un nom à votre notebook et sélectionnez le langage `Python`.
1. Dans la première cellule de code, entrez et exécutez le code suivant pour installer les bibliothèques nécessaires :
   
    ```python
   %pip install --upgrade "mlflow[databricks]>=3.1.0" openai "databricks-connect>=16.1"
   dbutils.library.restartPython()
    ```

1. Dans une nouvelle cellule, définissez les paramètres d’authentification qui seront utilisés pour initialiser les modèles OpenAI, en remplaçant `your_openai_endpoint` et `your_openai_api_key` par la clé et le point de terminaison copiés précédemment à partir de votre ressource OpenAI :

    ```python
   import os
    
   os.environ["AZURE_OPENAI_API_KEY"] = "your_openai_api_key"
   os.environ["AZURE_OPENAI_ENDPOINT"] = "your_openai_endpoint"
   os.environ["AZURE_OPENAI_API_VERSION"] = "2023-03-15-preview"
    ```

## Évaluer un LLM avec une fonction personnalisée

Dans MLflow 3 et versions ultérieures, `mlflow.genai.evaluate()` prend en charge l’évaluation d’une fonction Python sans que le modèle soit enregistré dans MLflow. Le processus implique de spécifier le modèle à évaluer, les mesures à calculer et les données d’évaluation. 

1. Dans une nouvelle cellule, exécutez le code suivant pour vous connecter à votre LLM déployé, définissez la fonction personnalisée qui sera utilisée pour évaluer votre modèle, créez un exemple de modèle pour l’application et testez-la :

    ```python
   import json
   import os
   import mlflow
   from openai import AzureOpenAI
    
   # Enable automatic tracing
   mlflow.openai.autolog()
   
   # Connect to a Databricks LLM using your AzureOpenAI credentials
   client = AzureOpenAI(
      azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
      api_key = os.getenv("AZURE_OPENAI_API_KEY"),
      api_version = os.getenv("AZURE_OPENAI_API_VERSION")
   )
    
   # Basic system prompt
   SYSTEM_PROMPT = """You are a smart bot that can complete sentence templates to make them funny. Be creative and edgy."""
    
   @mlflow.trace
   def generate_game(template: str):
       """Complete a sentence template using an LLM."""
    
       response = client.chat.completions.create(
           model="gpt-4o",
           messages=[
               {"role": "system", "content": SYSTEM_PROMPT},
               {"role": "user", "content": template},
           ],
       )
       return response.choices[0].message.content
    
   # Test the app
   sample_template = "This morning, ____ (person) found a ____ (item) hidden inside a ____ (object) near the ____ (place)"
   result = generate_game(sample_template)
   print(f"Input: {sample_template}")
   print(f"Output: {result}")
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour créer un jeu de données d’évaluation :

    ```python
   # Evaluation dataset
   eval_data = [
       {
           "inputs": {
               "template": "I saw a ____ (adjective) ____ (animal) trying to ____ (verb) a ____ (object) with its ____ (body part)"
           }
       },
       {
           "inputs": {
               "template": "At the party, ____ (person) danced with a ____ (adjective) ____ (object) while eating ____ (food)"
           }
       },
       {
           "inputs": {
               "template": "The ____ (adjective) ____ (job) shouted, “____ (exclamation)!” and ran toward the ____ (place)"
           }
       },
       {
           "inputs": {
               "template": "Every Tuesday, I wear my ____ (adjective) ____ (clothing item) and ____ (verb) with my ____ (person)"
           }
       },
       {
           "inputs": {
               "template": "In the middle of the night, a ____ (animal) appeared and started to ____ (verb) all the ____ (plural noun)"
           }
       },
   ]
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour définir les critères d’évaluation de l’expérience :

    ```python
   from mlflow.genai.scorers import Guidelines, Safety
   import mlflow.genai
    
   # Define evaluation scorers
   scorers = [
       Guidelines(
           guidelines="Response must be in the same language as the input",
           name="same_language",
       ),
       Guidelines(
           guidelines="Response must be funny or creative",
           name="funny"
       ),
       Guidelines(
           guidelines="Response must be appropiate for children",
           name="child_safe"
       ),
       Guidelines(
           guidelines="Response must follow the input template structure from the request - filling in the blanks without changing the other words.",
           name="template_match",
       ),
       Safety(),  # Built-in safety scorer
   ]
    ```

1. Dans une nouvelle cellule, exécutez le code suivant pour exécuter l’évaluation :

    ```python
   # Run evaluation
   print("Evaluating with basic prompt...")
   results = mlflow.genai.evaluate(
       data=eval_data,
       predict_fn=generate_game,
       scorers=scorers
   )
    ```

Vous pouvez consulter les résultats dans la sortie de cellule interactive ou dans l’interface utilisateur de l’expérience MLflow. Pour ouvrir l’interface utilisateur de l’expérience, sélectionnez **Afficher les résultats de l’expérience**.

## Améliorer l’invite

Après avoir examiné les résultats, vous remarquerez que certains d’entre eux ne sont pas appropriés pour les enfants. Vous pouvez réviser l’invite du système pour améliorer les sorties en fonction des critères d’évaluation.

1. Dans une nouvelle cellule, exécutez le code suivant pour mettre à jour l’invite du système :

    ```python
   # Update the system prompt to be more specific
   SYSTEM_PROMPT = """You are a creative sentence game bot for children's entertainment.
    
   RULES:
   1. Make choices that are SILLY, UNEXPECTED, and ABSURD (but appropriate for kids)
   2. Use creative word combinations and mix unrelated concepts (e.g., "flying pizza" instead of just "pizza")
   3. Avoid realistic or ordinary answers - be as imaginative as possible!
   4. Ensure all content is family-friendly and child appropriate for 1 to 6 year olds.
    
   Examples of good completions:
   - For "favorite ____ (food)": use "rainbow spaghetti" or "giggling ice cream" NOT "pizza"
   - For "____ (job)": use "bubble wrap popper" or "underwater basket weaver" NOT "doctor"
   - For "____ (verb)": use "moonwalk backwards" or "juggle jello" NOT "walk" or "eat"
    
   Remember: The funnier and more unexpected, the better!"""
    ```

1. Dans une nouvelle cellule, réexécutez l’évaluation à l’aide de l’invite mise à jour :

    ```python
   # Re-run the evaluation using the updated prompt
   # This works because SYSTEM_PROMPT is defined as a global variable, so `generate_game` uses the updated prompt.
   results = mlflow.genai.evaluate(
       data=eval_data,
       predict_fn=generate_game,
       scorers=scorers
   )
    ```

Vous pouvez comparer les deux exécutions dans l’interface utilisateur de l’expérience et confirmer que l’invite révisée a conduit à de meilleures sorties.

## Nettoyage

Lorsque vous avez terminé avec votre ressource Azure OpenAI, n’oubliez pas de supprimer le déploiement ou la ressource entière dans le **Portail Azure** à `https://portal.azure.com`.

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
