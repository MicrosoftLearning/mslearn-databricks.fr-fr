---
lab:
  title: "Déconseillé - Utiliser Apache\_Spark dans Azure\_Databricks"
---

# Utiliser Apache Spark dans Azure Databricks

Azure Databricks est une version basée sur Microsoft Azure de la plateforme Databricks open source reconnue. Azure Databricks repose sur Apache Spark et offre une solution hautement évolutive pour les tâches d’ingénierie et d’analyse des données qui impliquent l’utilisation de données dans des fichiers. L’un des avantages de Spark est la prise en charge d’un large éventail de langages de programmation, notamment Java, Scala, Python et SQL. Cela fait de Spark une solution très flexible pour les charges de travail de traitement des données, y compris le nettoyage et la manipulation des données, l’analyse statistique et le Machine Learning, ainsi que l’analytique données et la visualisation des données.

Cet exercice devrait prendre environ **45** minutes.

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
7. Attendez que le script se termine. Cette opération prend généralement environ 5 minutes, mais dans certains cas, elle peut être plus longue. Pendant que vous attendez, consultez l’article [Analyse exploratoire des données dans Azure Databricks](https://learn.microsoft.com/azure/databricks/exploratory-data-analysis/) dans la documentation Azure Databricks.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version 13.3 LTS ou ultérieure du runtime dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour effectuer cet exercice et ignorer cette procédure.

1. Dans le Portail Microsoft Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou le groupe de ressources contenant votre espace de travail Azure Databricks existant)
1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Mode cluster** : nœud unique
    - **Mode d’accès** : un seul utilisateur (*avec votre compte d’utilisateur sélectionné*)
    - **Version du runtime Databricks** : 13.3 LTS (Spark 3.4.1, Scala 2.12) ou version ultérieure
    - **Utiliser l’accélération photon** : sélectionné
    - **Type de nœud** : Standard_DS3_v2
    - **Arrêter après** *20* **minutes d’inactivité**

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Explorer les données à l’aide de Spark

Comme dans de nombreux environnements Spark, Databricks prend en charge l’utilisation de notebooks pour combiner des notes et des cellules de code interactives que vous pouvez utiliser pour explorer les données.

### Créer un notebook

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
1. Remplacez le nom du notebook par défaut (**Notebook sans titre *[date]***) par **Explorer les données avec Spark** et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

### Ingérer des données

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

    ```python
    %sh
    rm -r /dbfs/spark_lab
    mkdir /dbfs/spark_lab
    wget -O /dbfs/spark_lab/2019.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2019.csv
    wget -O /dbfs/spark_lab/2020.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2020.csv
    wget -O /dbfs/spark_lab/2021.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/2021.csv
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

### Interroger des données dans des fichiers

1. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Ensuite, dans la nouvelle cellule, entrez et exécutez le code suivant pour charger les données à partir des fichiers et afficher les 100 premières lignes.

    ```python
   df = spark.read.load('spark_lab/*.csv', format='csv')
   display(df.limit(100))
    ```

1. Affichez le résultat et notez que les données du fichier concernent les commandes clients, mais n'incluent pas les en-têtes de colonne ni les informations sur les types de données. Pour mieux comprendre les données, vous pouvez définir un *schéma* pour le dataframe.

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant, qui définit un schéma pour les données :

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   orderSchema = StructType([
        StructField("SalesOrderNumber", StringType()),
        StructField("SalesOrderLineNumber", IntegerType()),
        StructField("OrderDate", DateType()),
        StructField("CustomerName", StringType()),
        StructField("Email", StringType()),
        StructField("Item", StringType()),
        StructField("Quantity", IntegerType()),
        StructField("UnitPrice", FloatType()),
        StructField("Tax", FloatType())
   ])
   df = spark.read.load('/spark_lab/*.csv', format='csv', schema=orderSchema)
   display(df.limit(100))
    ```

1. Observez que cette fois-ci, le cadre de données comprend des en-têtes de colonne. Ajoutez ensuite une nouvelle cellule de code et utilisez-la pour exécuter le code suivant afin d'afficher les détails du schéma du cadre de données et de vérifier que les types de données corrects ont été appliqués :

    ```python
   df.printSchema()
    ```

### Filtrer un dataframe

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant, qui :
    - Filtre les colonnes du cadre de données des commandes clients pour n'inclure que le nom et l'adresse e-mail du client.
    - Compter le nombre total d’enregistrements de commande
    - Compter le nombre de clients distincts
    - Afficher les clients distincts

    ```python
   customers = df['CustomerName', 'Email']
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Observez les informations suivantes :

    - Lorsque vous effectuez une opération sur un dataframe, le résultat est un nouveau dataframe (dans ce cas, un nouveau dataframe customers est créé en sélectionnant un sous-ensemble spécifique de colonnes dans le dataframe df).
    - Les dataframes fournissent des fonctions telles que count et distinct qui peuvent être utilisées pour résumer et filtrer les données qu’ils contiennent.
    - La syntaxe `dataframe['Field1', 'Field2', ...]` offre un moyen rapide de définir un sous-ensemble de colonnes. Vous pouvez également utiliser la méthode **select**, pour que la première ligne du code ci-dessus puisse être écrite sous la forme `customers = df.select("CustomerName", "Email")`

1. Appliquons maintenant un filtre pour inclure uniquement les clients qui ont passé une commande pour un produit spécifique en exécutant le code suivant dans une nouvelle cellule de code :

    ```python
   customers = df.select("CustomerName", "Email").where(df['Item']=='Road-250 Red, 52')
   print(customers.count())
   print(customers.distinct().count())
   display(customers.distinct())
    ```

    Notez que vous pouvez « chaîner » plusieurs fonctions afin que la sortie d’une fonction devienne l’entrée de la suivante. Dans ce cas, le dataframe créé par la méthode select est le dataframe source de la méthode où utilisée pour appliquer des critères de filtrage.

### Agréger et regrouper des données dans un dataframe

1. Exécutez le code suivant dans une nouvelle cellule de code pour agréger et regrouper les données relatives aux commandes :

    ```python
   productSales = df.select("Item", "Quantity").groupBy("Item").sum()
   display(productSales)
    ```

    Remarquez que les résultats affichent la somme des quantités de commandes regroupées par produit. La méthode **groupBy** regroupe les lignes par *Item*, et la fonction d’agrégation **sum** suivante est appliquée à toutes les colonnes numériques restantes (dans ce cas, *Quantity*).

1. Dans une nouvelle cellule de code, essayons une autre agrégation :

    ```python
   yearlySales = df.select(year("OrderDate").alias("Year")).groupBy("Year").count().orderBy("Year")
   display(yearlySales)
    ```

    Cette fois, les résultats indiquent le nombre de commandes par an. Notez que la méthode select inclut une fonction **année** SQL pour extraire le composant année du champ *OrderDate*, puis une méthode **alias**est utilisée pour affecter un nom de colonne à la valeur d’année extraite. Les données sont ensuite regroupées par la colonne *Year* dérivée et le **nombre** de lignes dans chaque groupe est calculé avant que la méthode **orderBy** soit finalement utilisée pour trier le dataframe résultant.

> **Remarque** : Pour en savoir plus sur l’utilisation des Dataframes dans Azure Databricks, consultez [Introduction aux DataFrames : Python](https://docs.microsoft.com/azure/databricks/spark/latest/dataframes-datasets/introduction-to-dataframes-python) dans la documentation Azure Databricks.

### Interroger des données à l’aide de Spark SQL

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant :

    ```python
   df.createOrReplaceTempView("salesorders")
   spark_df = spark.sql("SELECT * FROM salesorders")
   display(spark_df)
    ```

    Les méthodes natives de l’objet dataframe que vous avez utilisé précédemment vous permettent d’interroger et d’analyser des données de manière très efficace. Toutefois, de nombreux analystes de données sont plus à l’aise avec la syntaxe SQL. Spark SQL est une API de langage SQL dans Spark que vous pouvez utiliser pour exécuter des instructions SQL, ou même pour conserver des données dans des tables relationnelles. Le code que vous venez d’exécuter crée une *vue* relationnelle des données dans un dataframe, puis utilise la bibliothèque **spark.sql** pour incorporer la syntaxe Spark SQL dans votre code Python et interroger la vue et retourner les résultats sous forme de dataframe.

### Exécuter du code SQL dans une cellule

1. Bien qu’il soit utile d’incorporer des instructions SQL dans une cellule contenant du code PySpark, les analystes de données veulent souvent simplement travailler directement dans SQL. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant.

    ```sql
   %sql
    
   SELECT YEAR(OrderDate) AS OrderYear,
          SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue
   FROM salesorders
   GROUP BY YEAR(OrderDate)
   ORDER BY OrderYear;
    ```

    Observez que :
    
    - La ligne ``%sql` au début de la cellule (appelée commande magique) indique que le runtime de langage Spark SQL doit être utilisé à la place de PySpark pour exécuter le code dans cette cellule.
    - Le code SQL fait référence à la vue **salesorder** que vous avez créée précédemment.
    - La sortie de la requête SQL s’affiche automatiquement en tant que résultat sous la cellule.
    
> **Remarque** : Pour plus d’informations sur Spark SQL et les dataframes, consultez la [documentation Spark SQL](https://spark.apache.org/docs/2.2.0/sql-programming-guide.html).

## Visualiser les données avec Spark

Selon le proverbe, une image vaut mille mots, et un graphique exprime souvent plus qu’un millier de lignes de données. Les notebooks dans Azure Databricks prennent en charge la visualisation des données à partir d’un dataframe ou d’une requête Spark SQL, mais elle n’est pas conçue en tant que graphique complet. Toutefois, vous pouvez utiliser les bibliothèques graphiques Python telles que matplotlib et seaborn pour créer des graphiques à partir de données dans des dataframes.

### Visualiser les résultats

1. Dans une nouvelle cellule de code, exécutez le code suivant pour interroger la table **salesorders** :

    ```sql
   %sql
    
   SELECT * FROM salesorders
    ```

1. Au-dessus du tableau des résultats, sélectionnez **+**, puis **Visualisation** pour afficher l’éditeur de visualisation et appliquer les options suivantes :
    - **Type de visualisation** : barre
    - **Colonne X** : Élément
    - **Colonne Y** : *Ajoutez une nouvelle colonne et sélectionnez* **Quantity**. *Appliquez* **l’agrégation** *Sum*.
    
1. Enregistrez la visualisation, puis réexécutez la cellule de code pour afficher le graphique résultant dans le carnet de notes.

### Bien démarrer avec matplotlib

1. Dans une nouvelle cellule de code, exécutez le code suivant pour récupérer des données de commandes clients dans un cadre de données :

    ```python
   sqlQuery = "SELECT CAST(YEAR(OrderDate) AS CHAR(4)) AS OrderYear, \
                   SUM((UnitPrice * Quantity) + Tax) AS GrossRevenue \
            FROM salesorders \
            GROUP BY CAST(YEAR(OrderDate) AS CHAR(4)) \
            ORDER BY OrderYear"
   df_spark = spark.sql(sqlQuery)
   df_spark.show()
    ```

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant, qui importe le **matplotlb** et l’utilise pour créer un graphique :

    ```python
   from matplotlib import pyplot as plt
    
   # matplotlib requires a Pandas dataframe, not a Spark one
   df_sales = df_spark.toPandas()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'])
   # Display the plot
   plt.show()
    ```

1. Passez en revue les résultats, qui se composent d’un histogramme indiquant le chiffre d’affaires brut total pour chaque année. Notez les fonctionnalités suivantes du code utilisé pour produire ce graphique :
    - La bibliothèque **matplotlib** nécessite un dataframe Pandas. Vous devez donc convertir le dataframe Spark retourné par la requête Spark SQL dans ce format.
    - Au cœur de la bibliothèque **matplotlib** figure l’objet **pyplot**. Il s’agit de la base de la plupart des fonctionnalités de traçage.

1. Les paramètres par défaut aboutissent à un graphique utilisable, mais il existe de nombreuses façons de le personnaliser. Ajoutez une nouvelle cellule de code avec le code suivant et exécutez-le :

    ```python
   # Clear the plot area
   plt.clf()
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Un tracé est techniquement contenu dans une **figure**. Dans les exemples précédents, la figure a été créée implicitement pour vous, mais vous pouvez la créer explicitement. Essayez d’exécuter ce qui suit dans une nouvelle cellule :

    ```python
   # Clear the plot area
   plt.clf()
   # Create a Figure
   fig = plt.figure(figsize=(8,3))
   # Create a bar plot of revenue by year
   plt.bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   # Customize the chart
   plt.title('Revenue by Year')
   plt.xlabel('Year')
   plt.ylabel('Revenue')
   plt.grid(color='#95a5a6', linestyle='--', linewidth=2, axis='y', alpha=0.7)
   plt.xticks(rotation=45)
   # Show the figure
   plt.show()
    ```

1. Une figure peut contenir plusieurs sous-tracés, chacun sur son propre axe. Utilisez ce code pour créer plusieurs graphiques :

    ```python
   # Clear the plot area
   plt.clf()
   # Create a figure for 2 subplots (1 row, 2 columns)
   fig, ax = plt.subplots(1, 2, figsize = (10,4))
   # Create a bar plot of revenue by year on the first axis
   ax[0].bar(x=df_sales['OrderYear'], height=df_sales['GrossRevenue'], color='orange')
   ax[0].set_title('Revenue by Year')
   # Create a pie chart of yearly order counts on the second axis
   yearly_counts = df_sales['OrderYear'].value_counts()
   ax[1].pie(yearly_counts)
   ax[1].set_title('Orders per Year')
   ax[1].legend(yearly_counts.keys().tolist())
   # Add a title to the Figure
   fig.suptitle('Sales Data')
   # Show the figure
   plt.show()
    ```

> **Remarque** : Pour en savoir plus sur le traçage avec matplotlib, consultez la [documentation matplotlib](https://matplotlib.org/).

### Utiliser la bibliothèque seaborn

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant, qui utilise la bibliothèque **seaborn** (qui est basée sur matplotlib et extrait certaines de sa complexité) pour créer un graphique :

    ```python
   import seaborn as sns
   
   # Clear the plot area
   plt.clf()
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. La bibliothèque **seaborn** simplifie la création de tracés complexes de données statistiques et vous permet de contrôler le thème visuel pour des visualisations de données cohérentes. Exécutez le code suivant dans une nouvelle cellule :

    ```python
   # Clear the plot area
   plt.clf()
   
   # Set the visual theme for seaborn
   sns.set_theme(style="whitegrid")
   
   # Create a bar chart
   ax = sns.barplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

1. Comme matplotlib. seaborn prend en charge plusieurs types de graphiques. Exécutez le code suivant pour créer un graphique en courbes :

    ```python
   # Clear the plot area
   plt.clf()
   
   # Create a bar chart
   ax = sns.lineplot(x="OrderYear", y="GrossRevenue", data=df_sales)
   plt.show()
    ```

> **Remarque** : Pour en savoir plus sur le traçage avec seaborn, consultez la [documentation seaborn](https://seaborn.pydata.org/index.html).

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.