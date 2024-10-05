---
lab:
  title: "Gérer un modèle Machine\_Learning à l’aide d’Azure\_Databricks"
---

# Gérer un modèle Machine Learning à l’aide d’Azure Databricks

L’entraînement d’un modèle Machine Learning à l’aide d’Azure Databricks implique l’utilisation d’une plateforme analytique unifiée qui fournit un environnement collaboratif pour le traitement des données, l’entraînement du modèle et le déploiement. Azure Databricks s’intègre à MLflow pour gérer le cycle de vie du Machine Learning, notamment le suivi des expériences et le service de modèle.

Cet exercice devrait prendre environ **20** minutes.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.
2. Utilisez le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, en sélectionnant un environnement ***PowerShell*** et en créant le stockage si vous y êtes invité. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Remarque** : si vous avez créé un shell cloud qui utilise un environnement *Bash*, utilisez le menu déroulant en haut à gauche du volet Cloud Shell pour le remplacer par ***PowerShell***.

3. Notez que vous pouvez redimensionner le volet Cloud Shell en faisant glisser la barre de séparation en haut du volet. Vous pouvez aussi utiliser les icônes **&#8212;** , **&#9723;** et **X** situées en haut à droite du volet pour réduire, agrandir et fermer le volet. Pour plus d’informations sur l’utilisation d’Azure Cloud Shell, consultez la [documentation Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Qu’est-ce que Databricks Machine Learning ?](https://learn.microsoft.com/azure/databricks/machine-learning/) dans la documentation Azure Databricks.

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
1. Remplacez le nom du notebook par défaut (**Notebook sans titre *[date]***) par **Machine Learning** et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

## Ingérer et préparer les données

Le scénario de cet exercice est basé sur des observations de manchots en Antarctique. L’objectif est de former un modèle Machine Learning pour prédire l’espèce d’un manchot observé en fonction de sa localisation et de ses mensurations corporelles.

> **Citation** : Le jeu de données sur les manchots utilisé dans cet exercice est un sous-ensemble des données collectées et publiées par [ Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) et la [station Palmer en Antarctique](https://pal.lternet.edu/), qui fait partie du [réseau mondial de recherche écologique à long terme (LTER)](https://lternet.edu/).

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes d’*interpréteur de commandes* pour télécharger les données relatives aux manchots à partir de GitHub dans le système de fichiers utilisé par votre cluster.

    ```bash
    %sh
    rm -r /dbfs/ml_lab
    mkdir /dbfs/ml_lab
    wget -O /dbfs/ml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
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
    
## Exécuter un pipeline pour prétraiter les données et entraîner un modèle ML

Avant d’entraîner votre modèle, vous devez suivre les étapes d’ingénierie de caractéristiques requises, puis adapter un algorithme aux données. Pour utiliser le modèle avec des données de test et générer ainsi des prédictions, vous devez appliquer les mêmes étapes d’ingénierie de caractéristiques aux données de test. Une manière plus efficace de construire et d’utiliser des modèles est d’encapsuler les transformateurs utilisés pour préparer les données et le modèle utilisé pour les entraîner dans un *pipeline*.

1. Utilisez le code suivant pour créer un pipeline qui encapsule les étapes de préparation des données et d’entraînement du modèle :

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import LogisticRegression
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model training algorithm steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = LogisticRegression(labelCol="Species", featuresCol="Features", maxIter=10, regParam=0.3)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Étant donné que les étapes d’ingénierie de caractéristiques sont désormais encapsulées dans le modèle entraîné par le pipeline, vous pouvez utiliser le modèle avec les données de test sans avoir à appliquer chaque transformation (elles seront appliquées automatiquement par le modèle).

1. Utilisez le code suivant pour appliquer le pipeline aux données de test et évaluer le modèle :

    ```python
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   display(predicted)

   # Generate evaluation metrics
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="Species", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Class metrics
   labels = [0,1,2]
   print("\nIndividual class metrics:")
   for label in sorted(labels):
       print ("Class %s" % (label))
   
       # Precision
       precision = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                       evaluator.metricName:"precisionByLabel"})
       print("\tPrecision:", precision)
   
       # Recall
       recall = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                                evaluator.metricName:"recallByLabel"})
       print("\tRecall:", recall)
   
       # F1 score
       f1 = evaluator.evaluate(prediction, {evaluator.metricLabel:label,
                                            evaluator.metricName:"fMeasureByLabel"})
       print("\tF1 Score:", f1)
   
   # Weighed (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1) 
    ```

## Inscrire et déployer le modèle

Vous avez déjà journalisé le modèle entraîné par chaque exécution d’expérience lors de l’exécution du pipeline. Vous pouvez également *inscrire* des modèles, et les déployer pour qu’ils soient mis à disposition d’applications clientes.

> **Remarque** : La mise à disposition de modèles est uniquement prise en charge dans les espaces de travail Azure Databricks *Premium*, et est limitée à [certaines régions](https://learn.microsoft.com/azure/databricks/resources/supported-regions).

1. Dans le volet gauche, sélectionnez **Expériences**.
1. Sélectionnez l’expérience générée avec le nom de votre notebook et affichez la page de détails de l’exécution de l’expérience la plus récente.
1. Utilisez le bouton **Inscrire le modèle** pour inscrire le modèle journalisé au cours de cette expérience. Quand vous y êtes invité, créez un modèle nommé **Penguin Predictor** (Outil de prédiction pour les manchots).
1. Une fois le modèle inscrit, visualisez la page **Modèles** (dans la barre de navigation à gauche), puis sélectionnez le modèle **Penguin Predictor**.
1. Dans la page du modèle **Penguin Predictor**, utilisez le bouton **Utiliser le modèle pour l’inférence** afin de créer un point de terminaison en temps réel avec les paramètres suivants :
    - **Modèle** : Penguin Predictor
    - **Version du modèle** : 1
    - **Point de terminaison** : predict-penguin
    - **Taille de calcul** : Small

    Le point de terminaison de mise à disposition est hébergé dans un nouveau cluster, dont la création peut prendre plusieurs minutes.
  
1. Une fois le point de terminaison créé, utilisez le bouton **Interroger le point de terminaison** en haut à droite pour ouvrir une interface à partir de laquelle vous pouvez tester le point de terminaison. Dans l’interface de test, sous l’onglet **Navigateur**, entrez la requête JSON suivante, puis utilisez le bouton **Envoyer la requête** pour appeler le point de terminaison et générer une prédiction.

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. Faites des essais en utilisant différentes valeurs relatives aux caractéristiques des manchots, puis observez les résultats retournés. Fermez ensuite l’interface de test.

## Supprimer le point de terminaison

Une fois que le point de terminaison n’est plus nécessaire, supprimez-le pour éviter des coûts inutiles.

Dans la page du point de terminaison **predict-penguin**, dans le menu **&#8285;**, sélectionnez **Supprimer**.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez fini de découvrir Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter des coûts Azure inutiles, et libérer de la capacité dans votre abonnement.

> **Informations complémentaires** : Pour plus d’informations, consultez la [documentation relative à Spark MLLib](https://spark.apache.org/docs/latest/ml-guide.html).
