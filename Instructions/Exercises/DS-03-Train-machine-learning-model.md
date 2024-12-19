---
lab:
  title: Prendre en main l’apprentissage automatique dans Azure Databricks
---

# Prendre en main l’apprentissage automatique dans Azure Databricks

Dans cet exercice, vous allez explorer les techniques de préparation des données et de formation des modèles Machine Learning dans Azure Databricks.

Cet exercice devrait prendre environ **45** minutes.

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

## Ingérer des données

Le scénario de cet exercice est basé sur des observations de manchots en Antarctique. L’objectif est d’entraîner un modèle Machine Learning pour prédire l’espèce d’un manchot observé sur la base de sa localisation et de ses mesures corporelles.

> **Citation** : Le jeu de données sur les manchots utilisé dans cet exercice est un sous-ensemble des données collectées et publiées par [ Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) et la [station Palmer en Antarctique](https://pal.lternet.edu/), qui fait partie du [réseau mondial de recherche écologique à long terme (LTER)](https://lternet.edu/).

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes d’*interpréteur de commandes* pour télécharger les données relatives aux manchots à partir de GitHub dans le système de fichiers utilisé par votre cluster.

    ```bash
    %sh
    rm -r /dbfs/ml_lab
    mkdir /dbfs/ml_lab
    wget -O /dbfs/ml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

## Explorer et nettoyer les données
  
Maintenant que vous avez ingéré le fichier de données, vous pouvez le charger dans un dataframe et l’afficher.

1. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Ensuite, dans la nouvelle cellule, entrez et exécutez le code suivant pour charger les données à partir des fichiers et les afficher.

    ```python
   df = spark.read.format("csv").option("header", "true").load("/ml_lab/penguins.csv")
   display(df)
    ```

    Le code lance les *travaux Spark* nécessaires pour charger les données, et la sortie est un objet *pyspark.sql.dataframe.DataFrame* nommé *df*. Vous verrez ces informations affichées directement sous le code, et vous pouvez utiliser le bouton **&#9656;** pour développer la sortie **df : pyspark.sql.dataframe.DataFrame** et voir les détails des colonnes qu’elle contient et leurs types de données. Étant donné que ces données ont été chargées à partir d’un fichier texte et contenaient des valeurs vides, Spark a affecté un type de données **string** à toutes les colonnes.
    
    Les données elles-mêmes se composent de mesures des détails suivants des manchots qui ont été observés en Antarctique :
    
    - **Island** : L’île en Antarctique où le manchot a été observé.
    - **CulmenLength** : La longueur en mm du culmen du manchot (bec).
    - **CulmenDepth** : La profondeur en mm du culmen du manchot.
    - **FlipperLength** : La longueur en mm de la nageoire du manchot.
    - **BodyMass** : La masse corporelle du manchot en grammes.
    - **Species** : Une valeur entière qui représente l’espèce du manchot :
      - **0** : *Adélie*
      - **1** : *Manchot papou*
      - **2** : *Manchot à jugulaire*
      
    Notre objectif dans ce projet est d’utiliser les caractéristiques observées d’un manchot (ses *caractéristiques*) afin de prédire son espèce (qui, dans la terminologie du Machine Learning, nous appelons l’*étiquette*).
      
    Notez que certaines observations contiennent des valeurs de données *null* ou « manquantes » pour certaines caractéristiques. Il n’est pas rare que les données sources brutes que vous ingérez présentent de tels problèmes. En général, la première étape d’un projet Machine Learning consiste à explorer les données de manière approfondie et à les nettoyer afin de les rendre plus adaptées à la formation d’un modèle Machine Learning.
    
1. Ajoutez une cellule et utilisez-la pour exécuter la cellule suivante pour supprimer les lignes avec des données incomplètes à l’aide de la méthode **dropna**, et pour appliquer les types de données appropriés aux données à l’aide de la méthode **select** avec les fonctions **col** et **astype**.

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   
   data = df.dropna().select(col("Island").astype("string"),
                              col("CulmenLength").astype("float"),
                             col("CulmenDepth").astype("float"),
                             col("FlipperLength").astype("float"),
                             col("BodyMass").astype("float"),
                             col("Species").astype("int")
                             )
   display(data)
    ```
    
    Une fois de plus, vous pouvez faire basculer les détails du dataframe retourné (cette fois nommé *données*) pour vérifier que les types de données ont été appliqués, et vous pouvez passer en revue les données pour vérifier que les lignes contenant des données incomplètes ont été supprimées.
    
    Dans un projet réel, vous devrez probablement effectuer davantage d’exploration et de nettoyage des données pour corriger (ou supprimer) les erreurs dans les données, identifier et supprimer les valeurs hors norme (des valeurs inhabituellement grandes ou petites) ou équilibrer les données afin qu’il y ait un nombre raisonnablement égal de lignes pour chaque étiquette que vous essayez de prédire.

    > **Conseil** : Vous pouvez en savoir plus sur les méthodes et les fonctions que vous pouvez utiliser avec les dataframes dans la [référence Spark SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html).

## Fractionner les données

Dans le cadre de cet exercice, nous partons du principe que les données sont désormais bien nettoyées et prêtes à être utilisées pour entraîner un modèle Machine Learning. L’étiquette que nous allons essayer de prédire est une catégorie ou une *classe* spécifique (l’espèce d’un manchot), donc le type de modèle Machine Learning que nous devons entraîner est un modèle de *classification*. La classification (ainsi que la *régression*, utilisée pour prédire une valeur numérique) est une forme de Machine Learning *supervisé* dans lequel nous utilisons des données d’apprentissage qui incluent des valeurs connues pour l’étiquette que nous voulons prédire. Le processus de formation d’un modèle consiste en fait à adapter un algorithme aux données afin de calculer la corrélation entre les valeurs des caractéristiques et la valeur connue de l’étiquette. Nous pouvons ensuite appliquer le modèle formé à une nouvelle observation pour laquelle nous connaissons uniquement les valeurs de la caractéristique et lui demander de prédire la valeur de l’étiquette.

Pour nous assurer que nous pouvons avoir confiance dans notre modèle formé, l’approche classique consiste à former le modèle avec seulement *certaines* données et à conserver d’autres données avec des valeurs d’étiquette connues que nous pouvons utiliser pour tester le modèle formé et voir la précision de ses prédictions. Pour atteindre cet objectif, nous allons fractionner le jeu de données complet en deux sous-ensembles aléatoires. Nous allons utiliser 70 % des données pour la formation et conserver 30 % pour les tests.

1. Ajoutez et exécutez une cellule de code avec le code suivant pour fractionner les données.

    ```python
   splits = data.randomSplit([0.7, 0.3])
   train = splits[0]
   test = splits[1]
   print ("Training Rows:", train.count(), " Testing Rows:", test.count())
    ```

## Effectuer l’ingénierie de caractéristiques

Après avoir nettoyé les données brutes, les scientifiques des données effectuent généralement un travail supplémentaire pour les préparer à la formation du modèle. Ce processus est couramment appelé *ingénierie de caractéristiques* et implique l’optimisation itérative des caractéristiques dans le jeu de données de formation pour produire le meilleur modèle possible. Les modifications spécifiques apportées aux caractéristiques dépendent des données et du modèle souhaité, mais il existe certaines tâches courantes d’ingénierie de caractéristiques que vous devez connaître.

### Encoder des caractéristiques de catégorie

Les algorithmes de Machine Learning sont généralement basés sur la recherche de relations mathématiques entre les caractéristiques et les étiquettes. Cela signifie qu’il est généralement préférable de définir les caractéristiques de vos données de formation en tant que valeurs *numériques*. Dans certains cas, vous pouvez avoir certaines caractéristiques qui sont *catégorielles* plutôt que numériques et qui sont exprimées sous forme de chaînes. Par exemple, le nom de l’île où l’observation de manchot s’est produite dans notre jeu de données. Toutefois, la plupart des algorithmes attendent des caractéristiques numériques. Ainsi, ces valeurs catégorielles basées sur des chaînes doivent être *encodées* en tant que nombres. Dans ce cas, nous allons utiliser un **StringIndexer** à partir de la bibliothèque **Spark MLLib** pour encoder le nom de l’île en tant que valeur numérique en affectant un index entier unique pour chaque nom d’île discret.

1. Exécutez le code suivant pour encoder les valeurs de colonne catégorielles **Island** en tant qu’index numériques.

    ```python
   from pyspark.ml.feature import StringIndexer

   indexer = StringIndexer(inputCol="Island", outputCol="IslandIdx")
   indexedData = indexer.fit(train).transform(train).drop("Island")
   display(indexedData)
    ```

    Dans les résultats, vous devez voir qu’au lieu d’un nom d’île, chaque ligne a maintenant une colonne **IslandIdx** avec une valeur entière représentant l’île sur laquelle l’observation a été enregistrée.

### Normaliser (mettre à l’échelle) des caractéristiques numériques

Intéressons-nous maintenant aux valeurs numériques de nos données. Ces valeurs (**CulmenLength**, **CulmenDepth**, **FlipperLength** et **BodyMass**) représentent toutes des mesures d’une sorte ou d’une autre, mais elles sont dans différentes échelles. Lors de la formation d’un modèle, les unités de mesure ne sont pas aussi importantes que les différences relatives entre différentes observations, et les caractéristiques représentées par des nombres plus importants peuvent souvent dominer l’algorithme de formation du modèle, ce qui fausse l’importance de la caractéristique lors du calcul d’une prédiction. Pour atténuer ce problème, il est courant de *normaliser* les valeurs numériques des caractéristiques afin qu’elles se situent toutes sur la même échelle relative (par exemple, une valeur décimale comprise entre 0,0 et 1,0).

Le code que nous utiliserons pour ce faire est un peu plus complexe que l’encodage catégoriel que nous avons effectué précédemment. Nous devons mettre à l’échelle les valeurs de plusieurs colonnes en même temps. La technique que nous utilisons consiste donc à créer une seule colonne contenant un *vecteur* (essentiellement un tableau) de toutes les caractéristiques numériques, puis à appliquer une échelle pour produire une nouvelle colonne de vecteurs avec les valeurs normalisées équivalentes.

1. Utilisez le code suivant pour normaliser les caractéristiques numériques et voir une comparaison des colonnes de vecteur pré-normalisées et normalisées.

    ```python
   from pyspark.ml.feature import VectorAssembler, MinMaxScaler

   # Create a vector column containing all numeric features
   numericFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   numericColVector = VectorAssembler(inputCols=numericFeatures, outputCol="numericFeatures")
   vectorizedData = numericColVector.transform(indexedData)
   
   # Use a MinMax scaler to normalize the numeric values in the vector
   minMax = MinMaxScaler(inputCol = numericColVector.getOutputCol(), outputCol="normalizedFeatures")
   scaledData = minMax.fit(vectorizedData).transform(vectorizedData)
   
   # Display the data with numeric feature vectors (before and after scaling)
   compareNumerics = scaledData.select("numericFeatures", "normalizedFeatures")
   display(compareNumerics)
    ```

    La colonne **numericFeatures** des résultats contient un vecteur pour chaque ligne. Le vecteur comprend quatre valeurs numériques non mise à l’échelle (les mesures d’origine du manchot). Vous pouvez utiliser le bouton **&#9656;** pour afficher plus clairement les valeurs discrètes.
    
    La colonne **normalizedFeatures** contient également un vecteur pour chaque observation de manchot, mais cette fois les valeurs du vecteur sont normalisées à une échelle relative en fonction des valeurs minimales et maximales pour chaque mesure.

### Préparer des caractéristiques et des étiquettes pour la formation

Maintenant, rassemblons tout et créons une colonne unique contenant toutes les caractéristiques (le nom catégorique codé de l’île et les mesures normalisées des manchots), et une autre colonne contenant l’étiquette de classe pour laquelle nous voulons former un modèle prédictif (l’espèce de manchot).

1. Exécutez le code ci-dessous :

    ```python
   featVect = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="featuresVector")
   preppedData = featVect.transform(scaledData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   display(preppedData)
    ```

    Le vecteur de** caractéristiques** contient cinq valeurs (la valeur encodée de l’île et les valeurs normalisées de la longueur du culmen, la profondeur du culmen, la longueur de la nageoire et la masse corporelle). L’étiquette contient un code entier simple qui indique la classe des espèces de manchots.

## Entraîner un modèle Machine Learning

Maintenant que les données de formation sont préparées, vous pouvez l’utiliser pour entraîner un modèle. Les modèles sont entraînés à l’aide d’un *algorithme* qui tente d’établir une relation entre les caractéristiques et les étiquettes. Étant donné que, dans ce cas, vous souhaitez entraîner un modèle qui prédit une catégorie de *classe*, vous devez utiliser un algorithme de *classification*. Il existe de nombreux algorithmes de classification. Commençons par un algorithme bien établi : la régression logistique, qui tente itérativement de trouver les coefficients optimaux qui peuvent être appliqués aux données des caractéristiques dans un calcul logistique qui prédit la probabilité pour chaque valeur d’étiquette de classe. Pour entraîner le modèle, vous devez adapter l’algorithme de régression logistique aux données d’entraînement.

1. Exécutez le code suivant pour entraîner un modèle.

    ```python
   from pyspark.ml.classification import LogisticRegression

   lr = LogisticRegression(labelCol="label", featuresCol="features", maxIter=10, regParam=0.3)
   model = lr.fit(preppedData)
   print ("Model trained!")
    ```

    La plupart des algorithmes prennent en charge les paramètres qui vous donnent un certain contrôle sur la façon dont le modèle est entraîné. Dans ce cas, l’algorithme de régression logistique vous demande d’identifier la colonne contenant le vecteur de caractéristiques et la colonne contenant l’étiquette connue ; il vous permet également de spécifier le nombre maximum d’itérations effectuées pour trouver les coefficients optimaux pour le calcul logistique, ainsi qu’un paramètre de régularisation qui est utilisé pour empêcher le modèle de *surajuster* (en d’autres termes, établir un calcul logistique qui fonctionne bien avec les données d’apprentissage, mais qui ne se généralise pas bien lorsqu’il est appliqué à de nouvelles données).

## Tester le modèle

Maintenant que vous avez un modèle entraîné, vous pouvez le tester avec les données que vous avez conservées. Avant de pouvoir effectuer cette opération, vous devez effectuer les mêmes transformations d’ingénierie de caractéristiques pour les données de test que celles que vous avez appliquées aux données d’entraînement (dans ce cas, encoder le nom de l’île et normaliser les mesures). Ensuite, vous pouvez utiliser le modèle pour prédire les étiquettes des caractéristiques des données de test et comparer les étiquettes prédites aux étiquettes connues.

1. Utilisez le code suivant pour préparer les données de test, puis générer des prédictions :

    ```python
   # Prepare the test data
   indexedTestData = indexer.fit(test).transform(test).drop("Island")
   vectorizedTestData = numericColVector.transform(indexedTestData)
   scaledTestData = minMax.fit(vectorizedTestData).transform(vectorizedTestData)
   preppedTestData = featVect.transform(scaledTestData)[col("featuresVector").alias("features"), col("Species").alias("label")]
   
   # Get predictions
   prediction = model.transform(preppedTestData)
   predicted = prediction.select("features", "probability", col("prediction").astype("Int"), col("label").alias("trueLabel"))
   display(predicted)
    ```

    Les résultats incluent les colonnes suivantes :
    
    - **caractéristiques** : Les données des caractéristiques préparées du jeu de données de test.
    - **probabilité** : La probabilité calculée par le modèle pour chaque classe. Il s’agit d’un vecteur contenant trois valeurs de probabilité (car il existe trois classes) qui s’additionnent pour donner un total de 1,0 (il est supposé qu’il y a une probabilité de 100 % que le manchot appartienne à *l’une* des trois classes d’espèces).
    - **Prédiction** : L’étiquette de classe prédite (celle avec la probabilité la plus élevée).
    - **trueLabel** : Valeur d’étiquette connue réelle des données de test.
    
    Pour évaluer l’efficacité du modèle, vous pouvez simplement comparer les étiquettes prédites et les vraies étiquettes des résultats. Toutefois, vous pouvez obtenir des métriques plus significatives à l’aide d’un évaluateur de modèle. Dans ce cas, utilisez un évaluateur de classification multiclasse (car il existe plusieurs étiquettes de classe possibles).

1. Utilisez le code suivant pour obtenir des métriques d’évaluation pour un modèle de classification en fonction des résultats des données de test :

    ```python
   from pyspark.ml.evaluation import MulticlassClassificationEvaluator
   
   evaluator = MulticlassClassificationEvaluator(labelCol="label", predictionCol="prediction")
   
   # Simple accuracy
   accuracy = evaluator.evaluate(prediction, {evaluator.metricName:"accuracy"})
   print("Accuracy:", accuracy)
   
   # Individual class metrics
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
   
   # Weighted (overall) metrics
   overallPrecision = evaluator.evaluate(prediction, {evaluator.metricName:"weightedPrecision"})
   print("Overall Precision:", overallPrecision)
   overallRecall = evaluator.evaluate(prediction, {evaluator.metricName:"weightedRecall"})
   print("Overall Recall:", overallRecall)
   overallF1 = evaluator.evaluate(prediction, {evaluator.metricName:"weightedFMeasure"})
   print("Overall F1 Score:", overallF1)
    ```

    Les métriques d’évaluation calculées pour la classification multiclasse sont les suivantes :
    
    - **Justesse** : La proportion des prédictions globales correctes.
    - Métriques par classe :
      - **Précision** : La proportion des prédictions de cette classe qui étaient correctes.
      - **Rappel** : La proportion des instances réelles de cette classe qui ont été correctement prédites.
      - **Score F1** : Une métrique combinée pour la précision et le rappel
    - Métriques combinées (pondérées) de précision, rappel et F1 pour toutes les classes.
    
    > **Remarque** : Il peut sembler, à première vue, que la mesure de la précision globale constitue le meilleur moyen d’évaluer la performance prédictive d’un modèle. Cependant, il faut tenir compte de ce qui suit. Supposons que les manchots papou constituent 95 % de la population des manchots dans votre lieu d’étude. Un modèle qui prédit toujours l’étiquette **1** (la classe des manchots papou) aura une précision de 0,95. Cela ne signifie pas qu’il s’agit d’un excellent modèle pour prédire une espèce de manchot en fonction des caractéristiques ! C’est pourquoi les scientifiques des données ont tendance à explorer des métriques supplémentaires pour mieux comprendre comment un modèle de classification prédit pour chaque étiquette de classe possible.

## Utiliser un pipeline

Vous avez entraîné votre modèle en effectuant les étapes d’ingénierie de caractéristiques requises, puis en ajustant un algorithme aux données. Pour utiliser le modèle avec des données de test pour générer des prédictions (appelées *inférences*), vous devez appliquer les mêmes étapes d’ingénierie de caractéristiques aux données de test. Une manière plus efficace de construire et d’utiliser des modèles est d’encapsuler les transformateurs utilisés pour préparer les données et le modèle utilisé pour les entraîner dans un *pipeline*.

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

1. Utilisez le code suivant pour appliquer le pipeline aux données de test :

    ```python
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   display(predicted)
    ```

## Essayer un autre algorithme

Jusqu’à présent, vous avez entraîné un modèle de classification à l’aide de l’algorithme de régression logistique. Modifions cette étape dans le pipeline pour essayer un autre algorithme.

1. Exécutez le code suivant pour créer un pipeline qui utilise un algorithme d’arbre de décision :

    ```python
   from pyspark.ml import Pipeline
   from pyspark.ml.feature import StringIndexer, VectorAssembler, MinMaxScaler
   from pyspark.ml.classification import DecisionTreeClassifier
   
   catFeature = "Island"
   numFeatures = ["CulmenLength", "CulmenDepth", "FlipperLength", "BodyMass"]
   
   # Define the feature engineering and model steps
   catIndexer = StringIndexer(inputCol=catFeature, outputCol=catFeature + "Idx")
   numVector = VectorAssembler(inputCols=numFeatures, outputCol="numericFeatures")
   numScaler = MinMaxScaler(inputCol = numVector.getOutputCol(), outputCol="normalizedFeatures")
   featureVector = VectorAssembler(inputCols=["IslandIdx", "normalizedFeatures"], outputCol="Features")
   algo = DecisionTreeClassifier(labelCol="Species", featuresCol="Features", maxDepth=10)
   
   # Chain the steps as stages in a pipeline
   pipeline = Pipeline(stages=[catIndexer, numVector, numScaler, featureVector, algo])
   
   # Use the pipeline to prepare data and fit the model algorithm
   model = pipeline.fit(train)
   print ("Model trained!")
    ```

    Cette fois, le pipeline inclut les mêmes étapes de préparation des caractéristiques que précédemment, mais utilise un algorithme *d’arbre de décision* pour entraîner le modèle.
    
   1. Exécutez le code suivant pour utiliser le nouveau pipeline avec les données de test :

    ```python
   # Get predictions
   prediction = model.transform(test)
   predicted = prediction.select("Features", "probability", col("prediction").astype("Int"), col("Species").alias("trueLabel"))
   
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

## Enregistrer le modèle

En réalité, vous essayez d’entraîner de manière itérative le modèle avec différents algorithmes (et paramètres) pour trouver le meilleur modèle pour vos données. Pour l’instant, nous allons nous tenir au modèle d’arbres de décision que nous avons formé. Nous allons l’enregistrer afin que nous puissions l’utiliser plus tard avec d’autres observations de manchot.

1. Utilisez le code suivant pour enregistrer le modèle :

    ```python
   model.save("/models/penguin.model")
    ```

    Maintenant, si vous sortez et repérez un nouveau manchot, vous pouvez charger le modèle sauvegardé et l’utiliser pour prédire l’espèce du manchot sur la base de vos mesures de ses caractéristiques. L’utilisation d’un modèle pour générer des prédictions à partir de nouvelles données est appelée *inférence*.

1. Exécutez le code suivant pour charger le modèle et l’utiliser pour prédire l’espèce d'un nouveau manchot observé :

    ```python
   from pyspark.ml.pipeline import PipelineModel

   persistedModel = PipelineModel.load("/models/penguin.model")
   
   newData = spark.createDataFrame ([{"Island": "Biscoe",
                                     "CulmenLength": 47.6,
                                     "CulmenDepth": 14.5,
                                     "FlipperLength": 215,
                                     "BodyMass": 5400}])
   
   
   predictions = persistedModel.transform(newData)
   display(predictions.select("Island", "CulmenDepth", "CulmenLength", "FlipperLength", "BodyMass", col("prediction").alias("PredictedSpecies")))
    ```

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez fini de découvrir Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter des coûts Azure inutiles, et libérer de la capacité dans votre abonnement.

> **Informations complémentaires** : Pour plus d’informations, consultez la [documentation relative à Spark MLLib](https://spark.apache.org/docs/latest/ml-guide.html).
