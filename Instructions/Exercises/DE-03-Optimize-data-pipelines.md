---
lab:
  title: "Optimiser les pipelines de données pour améliorer les performances dans Azure\_Databricks"
---

# Optimiser les pipelines de données pour améliorer les performances dans Azure Databricks

L’optimisation des pipelines de données dans Azure Databricks peut améliorer considérablement les performances et l’efficacité. L’utilisation du chargeur automatique pour l’ingestion de données incrémentielles, associée à la couche de stockage de Delta Lake, garantit la fiabilité et les transactions ACID. L’implémentation du salage peut empêcher l’asymétrie des données, tandis que le clustering de l’ordre de plan optimise les lectures de fichiers en associant des informations connexes. Les fonctionnalités de réglage automatique d’Azure Databricks et l’optimiseur basé sur les coûts peuvent améliorer les performances en ajustant les paramètres en fonction des besoins en charge de travail.

Ce labo prend environ **30** minutes.

> **Note** : l’interface utilisateur Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.
2. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***PowerShell***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Note** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *Bash*, basculez-le vers ***PowerShell***.

3. Notez que vous pouvez redimensionner Cloud Shell en faisant glisser la barre de séparation en haut du volet. Vous pouvez aussi utiliser les icônes **&#8212;**, **&#10530;** et **X** situées en haut à droite du volet pour réduire, agrandir et fermer ce dernier. Pour plus d’informations sur l’utilisation d’Azure Cloud Shell, consultez la [documentation Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

4. Dans le volet PowerShell, entrez les commandes suivantes pour cloner ce référentiel :

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. Une fois le référentiel cloné, entrez la commande suivante pour exécuter le script **setup.ps1**, qui approvisionne un espace de travail Azure Databricks dans une région disponible :

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. Si vous y êtes invité, choisissez l’abonnement à utiliser (uniquement si vous avez accès à plusieurs abonnements Azure).

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Présentation de Delta Lake](https://docs.microsoft.com/azure/databricks/delta/delta-intro) dans la documentation Azure Databricks.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version 13.3 LTS ou ultérieure du runtime dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour effectuer cet exercice et ignorer cette procédure.

1. Dans le Portail Microsoft Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou le groupe de ressources contenant votre espace de travail Azure Databricks existant)

1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).

1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**. Vous devrez peut-être consulter le sous-menu **Plus**.

1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Mode cluster** : nœud unique
    - **Mode d’accès** : un seul utilisateur (*avec votre compte d’utilisateur sélectionné*)
    - **Version du runtime Databricks** : 13.3 LTS (Spark 3.4.1, Scala 2.12) ou version ultérieure
    - **Utiliser l’accélération photon** : sélectionné
    - **Type de nœud** : Standard_D4ds_v5
    - **Arrêter après** *20* **minutes d’inactivité**

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

    > **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Créer un notebook et ingérer des données

1. Dans la barre latérale, utilisez le lien **(+) Nouveau** pour créer un **notebook** et modifier le nom du notebook par défaut (**Notebook sans titre *[date]***) en **Optimiser l’ingestion des données**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

2. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-01.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-01.parquet
     ```

3. Sous la sortie de la première cellule, utilisez l’icône **+ Code** pour ajouter une nouvelle cellule et exécutez le code suivant dans celle-ci pour charger le jeu de données dans un DataFrame :
   
     ```python
    # Load the dataset into a DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/yellow_tripdata_2021-01.parquet")
    display(df)
     ```

4. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

## Optimisez l’ingestion des données avec le chargeur automatique :

L’optimisation de l’ingestion des données est essentielle pour gérer efficacement les jeux de données volumineux. Le chargeur automatique est conçu pour traiter les nouveaux fichiers de données à mesure qu’ils arrivent dans le stockage cloud, en prenant en charge différents formats de fichiers et services de stockage cloud. 

Auto Loader fournit une source de flux structuré appelée `cloudFiles`. À partir du chemin d’accès du répertoire d’entrée sur le stockage de fichiers dans le cloud, la source `cloudFiles` traite automatiquement les nouveaux fichiers à mesure qu’ils arrivent, avec la possibilité de traiter également les fichiers existants dans ce répertoire. 

1. Dans une nouvelle cellule, exécutez le code suivant pour créer un flux en fonction du dossier contenant les exemples de données :

     ```python
     df = (spark.readStream
             .format("cloudFiles")
             .option("cloudFiles.format", "parquet")
             .option("cloudFiles.schemaLocation", "/stream_data/nyc_taxi_trips/schema")
             .load("/nyc_taxi_trips/"))
     df.writeStream.format("delta") \
         .option("checkpointLocation", "/stream_data/nyc_taxi_trips/checkpoints") \
         .option("mergeSchema", "true") \
         .start("/delta/nyc_taxi_trips")
     display(df)
     ```

2. Dans une nouvelle cellule, exécutez le code suivant pour ajouter un nouveau fichier Parquet au flux :

     ```python
    %sh
    rm -r /dbfs/nyc_taxi_trips
    mkdir /dbfs/nyc_taxi_trips
    wget -O /dbfs/nyc_taxi_trips/yellow_tripdata_2021-02_edited.parquet https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/yellow_tripdata_2021-02_edited.parquet
     ```
   
    Le nouveau fichier a une nouvelle colonne. Le flux s’arrête donc avec une erreur `UnknownFieldException`. Avant que votre stream ne génère cette erreur, Auto Loader effectue l’inférence de schéma sur le dernier micro-batch de données et met à jour l’emplacement du schéma avec le schéma le plus récent en fusionnant les nouvelles colonnes à la fin du schéma. Les types de données des colonnes existantes restent inchangés.

3. Réexécutez la cellule de code de diffusion en continu et vérifiez que deux nouvelles colonnes (**new_column** et *_rescued_data**) ont été ajoutées à la table : La colonne **_rescued_data** contient toutes les données qui ne sont pas analysées en raison d’une incompatibilité de type, d’une incompatibilité de casse ou d’une colonne manquante dans le schéma.

4. Sélectionnez **Interrompre** pour arrêter la diffusion en continu des données.
   
    Les données de diffusion sont écrites dans les tables Delta. Delta Lake fournit un ensemble d’améliorations sur les fichiers Parquet traditionnels, comme les transactions ACID, l’évolution du schéma et le voyage dans le temps. Il unifie le traitement des données de diffusion et de traitement des données par lots, ce qui en fait une solution puissante pour la gestion des charges de travail des Big Data.

## Optimiser la transformation des données

L’asymétrie des données est un défi important dans l’informatique distribuée, en particulier dans le traitement des Big Data avec des infrastructures telles qu’Apache Spark. Le salage est une technique efficace pour optimiser l’asymétrie des données en ajoutant un composant aléatoire, ou « sel », aux clés avant le partitionnement. Ce processus permet de distribuer des données plus uniformément entre les partitions, ce qui entraîne une charge de travail plus équilibrée et des performances améliorées.

1. Dans une nouvelle cellule, exécutez le code suivant pour décomposer une partition asymétrique volumineuse en partitions plus petites en ajoutant une colonne de *sel* avec des entiers aléatoires :

     ```python
    from pyspark.sql.functions import lit, rand

    # Convert streaming DataFrame back to batch DataFrame
    df = spark.read.parquet("/nyc_taxi_trips/*.parquet")
     
    # Add a salt column
    df_salted = df.withColumn("salt", (rand() * 100).cast("int"))

    # Repartition based on the salted column
    df_salted.repartition("salt").write.format("delta").mode("overwrite").save("/delta/nyc_taxi_trips_salted")

    display(df_salted)
     ```   

## Optimiser le stockage

Delta Lake offre une suite de commandes d’optimisation qui peuvent améliorer considérablement les performances et la gestion du stockage de données. La commande `optimize` est conçue pour améliorer la vitesse des requêtes en organisant les données plus efficacement par le biais de techniques telles que le compactage et l’ordre de plan.

Le compactage consolide les fichiers plus petits en fichiers plus volumineux, ce qui peut être particulièrement bénéfique pour les requêtes de lecture. L’ordre de plan implique l’organisation de points de données afin que les informations associées soient stockées ensemble, ce qui réduit le temps nécessaire pour accéder à ces données pendant les requêtes.

1. Dans une nouvelle cellule, exécutez le code suivant pour effectuer un compactage dans la table Delta :

     ```python
    from delta.tables import DeltaTable

    delta_table = DeltaTable.forPath(spark, "/delta/nyc_taxi_trips")
    delta_table.optimize().executeCompaction()
     ```

2. Dans une nouvelle cellule, exécutez le code suivant pour effectuer le clustering de l’ordre de plan :

     ```python
    delta_table.optimize().executeZOrderBy("tpep_pickup_datetime")
     ```

Cette technique colocalise les informations associées dans le même ensemble de fichiers, ce qui améliore les performances des requêtes.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
