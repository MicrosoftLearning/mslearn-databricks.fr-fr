---
lab:
  title: Optimiser les hyperparamètres pour le Machine Learning dans Azure Databricks
---

# Optimiser les hyperparamètres pour le Machine Learning dans Azure Databricks

Dans cet exercice, vous allez utiliser la bibliothèque **Hyperopt** afin d’optimiser les hyperparamètres pour l’entraînement d’un modèle Machine Learning dans Azure Databricks.

Cet exercice devrait prendre environ **30** minutes.

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

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. Une fois le référentiel cloné, entrez la commande suivante pour exécuter le script **setup.ps1**, qui approvisionne un espace de travail Azure Databricks dans une région disponible :

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. Si vous y êtes invité, choisissez l’abonnement à utiliser (uniquement si vous avez accès à plusieurs abonnements Azure).
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [paramétrage des hyperparamètres](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) dans la documentation Azure Databricks.

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

## Créer un notebook

Vous allez exécuter du code qui utilise la bibliothèque Spark MLLib pour entraîner un modèle Machine Learning. La première étape consiste donc à créer un notebook dans votre espace de travail.

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
1. Remplacez le nom du notebook par défaut (**Notebook sans titre *[date]***) par **paramétrage des hyperparamètres** et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

## Ingérer des données

Le scénario de cet exercice est basé sur des observations de manchots en Antarctique. L’objectif est d’entraîner un modèle Machine Learning pour prédire l’espèce d’un manchot observé sur la base de sa localisation et de ses mesures corporelles.

> **Citation** : Le jeu de données sur les manchots utilisé dans cet exercice est un sous-ensemble des données collectées et publiées par [ Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) et la [station Palmer en Antarctique](https://pal.lternet.edu/), qui fait partie du [réseau mondial de recherche écologique à long terme (LTER)](https://lternet.edu/).

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes d’*interpréteur de commandes* pour télécharger les données relatives aux manchots à partir de GitHub dans le système de fichiers utilisé par votre cluster.

    ```bash
    %sh
    rm -r /dbfs/hyperopt_lab
    mkdir /dbfs/hyperopt_lab
    wget -O /dbfs/hyperopt_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.
1. Préparez maintenant les données pour le Machine Learning. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Ensuite, dans la nouvelle cellule, entrez et exécutez le code suivant pour :
    - Supprimer toutes les lignes incomplètes
    - Appliquer les types de données appropriés
    - Afficher un échantillon aléatoire des données
    - Fractionnez les données en deux jeux de données : un pour l’entraînement et un autre pour les tests.


    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = spark.read.format("csv").option("header", "true").load("/hyperopt_lab/penguins.csv")
   data = data.dropna().select(col("Island").astype("string"),
                             col("CulmenLength").astype("float"),
                             col("CulmenDepth").astype("float"),
                             col("FlipperLength").astype("float"),
                             col("BodyMass").astype("float"),
                             col("Species").astype("int")
                             )
   display(data.sample(0.2))
   
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Optimiser les valeurs d’hyperparamètres pour l’entraînement d’un modèle

Vous entraînez un modèle Machine Learning en ajustant les fonctionnalités à un algorithme qui calcule l’étiquette la plus probable. Les algorithmes prennent les données d’apprentissage en tant que paramètre et tentent de calculer une relation mathématique entre les fonctionnalités et les étiquettes. En plus des données, la plupart des algorithmes utilisent un ou plusieurs *hyperparamètres* pour influencer la façon dont la relation est calculée. La détermination des valeurs optimales des hyperparamètres est une partie importante du processus itératif d'entraînement du modèle.

Pour vous aider à déterminer des valeurs d’hyperparamètres optimales, Azure Databricks prend en charge **Hyperopt**, une bibliothèque qui vous permet d’essayer plusieurs valeurs d’hyperparamètres et de trouver la meilleure combinaison pour vos données.

La première étape de l’utilisation d’Hyperopt consiste à créer une fonction qui :

- effectue l’apprentissage d’un modèle à l’aide d’une ou plusieurs valeurs d’hyperparamètres passées à la fonction en tant que paramètres ;
- calcule une métrique de performance qui peut être utilisée pour mesurer la *perte* (à quel point le modèle s'éloigne de la performance de prédiction parfaite) ;
- retourne la valeur de perte afin qu’elle puisse être optimisée (réduite) de manière itérative en essayant différentes valeurs d’hyperparamètres.

1. Ajoutez une nouvelle cellule et utilisez le code suivant pour créer une fonction qui utilise les données sur les manchots pour entraîner un modèle de classification qui prédit l'espèce d'un manchot en fonction de son emplacement et de ses mesures :

    ```python
   from hyperopt import STATUS_OK
   import mlflow
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   def objective(params):
       # Train a model using the provided hyperparameter value
       catFeature = "Island"
       numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
       catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
       numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
       numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
       featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
       mlAlgo = DecisionTreeClassifier(labelCol="Species",    
                                       featuresCol="Features",
                                       maxDepth=params['MaxDepth'], maxBins=params['MaxBins'])
       pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, mlAlgo])
       model = pipeline.fit(train)
       
       # Evaluate the model to get the target metric
       prediction = model.transform(test)
       eval = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction", metricName="accuracy")
       accuracy = eval.evaluate(prediction)
       
       # Hyperopt tries to minimize the objective function, so you must return the negative accuracy.
       return {'loss': -accuracy, 'status': STATUS_OK}
    ```

1. Ajoutez une nouvelle cellule et utilisez le code suivant pour :
    - définir un espace de recherche qui spécifie la plage de valeurs à utiliser pour un ou plusieurs hyperparamètres (pour plus d’informations, consultez [Définition d’un espace de recherche](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/) dans la documentation Hyperopt) ;
    - spécifier l’algorithme Hyperopt que vous souhaitez utiliser (pour plus d’informations, consultez [Algorithmes](http://hyperopt.github.io/hyperopt/#algorithms) dans la documentation Hyperopt) ;
    - utiliser la fonction **hyperopt.fmin** pour appeler votre fonction d’entraînement à plusieurs reprises et essayer de réduire la perte.

    ```python
   from hyperopt import fmin, tpe, hp
   
   # Define a search space for two hyperparameters (maxDepth and maxBins)
   search_space = {
       'MaxDepth': hp.randint('MaxDepth', 10),
       'MaxBins': hp.choice('MaxBins', [10, 20, 30])
   }
   
   # Specify an algorithm for the hyperparameter optimization process
   algo=tpe.suggest
   
   # Call the training function iteratively to find the optimal hyperparameter values
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=6)
   
   print("Best param values: ", argmin)
    ```

1. Observez comment le code exécute de manière itérative la fonction d'entraînement 6 fois (en fonction du paramètre **max_evals**). Chaque exécution est enregistrée par MLflow et vous pouvez utiliser le bouton de bascule **&#9656 ;** pour développer la sortie de l’exécution **MLflow** sous la cellule de code et sélectionner le lien hypertexte de l’**expérience** pour les afficher. Chaque exécution se voit attribuer un nom aléatoire, et vous pouvez les voir dans la visionneuse d'exécution MLflow pour consulter les détails des paramètres et des métriques qui ont été enregistrés.
1. Une fois que toutes les exécutions sont terminées, observez que le code affiche les détails des meilleures valeurs d'hyperparamètres trouvées (la combinaison qui a entraîné la perte la plus faible). Dans ce cas, le paramètre **MaxBins** est défini comme un choix parmi une liste de trois valeurs possibles (10, 20 et 30). La meilleure valeur indique l’élément de base zéro dans la liste (donc 0=10, 1=20 et 2=30). Le paramètre **MaxDepth** est défini comme un entier aléatoire compris entre 0 et 10, et la valeur entière qui a donné le meilleur résultat est affichée. Pour plus d’informations sur la spécification d’étendues de valeurs hyperparamètres pour les espaces de recherche, consultez [expressions de paramètre](http://hyperopt.github.io/hyperopt/getting-started/search_spaces/#parameter-expressions) dans la documentation Hyperopt.

## Utiliser la classe Trials pour journaliser les détails de l’exécution

Outre l’utilisation des exécutions d’expériences MLflow pour journaliser les détails de chaque itération, vous pouvez également utiliser la classe **hyperopt.Trials** pour enregistrer et afficher les détails de chaque exécution.

1. Ajoutez une nouvelle cellule et utilisez le code suivant pour afficher les détails de chaque exécution enregistrée par la classe **Trial** :

    ```python
   from hyperopt import Trials
   
   # Create a Trials object to track each run
   trial_runs = Trials()
   
   argmin = fmin(
     fn=objective,
     space=search_space,
     algo=algo,
     max_evals=3,
     trials=trial_runs)
   
   print("Best param values: ", argmin)
   
   # Get details from each trial run
   print ("trials:")
   for trial in trial_runs.trials:
       print ("\n", trial)
    ```

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé l’exploration d’Azure Databricks, vous pouvez supprimer les ressources que vous avez créées afin d’éviter des coûts Azure non nécessaires et de libérer de la capacité dans votre abonnement.

> **Informations complémentaires** : Pour plus d’informations, consultez [paramétrage d’hyperparamètres](https://learn.microsoft.com/azure/databricks/machine-learning/automl-hyperparam-tuning/) dans la documentation Azure Databricks.