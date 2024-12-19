---
lab:
  title: Utiliser Delta Lake dans Azure Databricks
---

# Utiliser Delta Lake dans Azure Databricks

Delta Lake est un projet open source permettant de créer une couche de stockage de données transactionnelles pour Spark au-dessus d’un lac de données. Delta Lake ajoute la prise en charge de la sémantique relationnelle pour les opérations de données par lots et de streaming, et permet la création d’une architecture *Lakehouse*, dans laquelle Apache Spark peut être utilisé pour traiter et interroger des données dans des tables basées sur des fichiers sous-jacents dans le lac de données.

Ce labo prend environ **30** minutes.

> **Remarque** : l’interface utilisateur d’Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.
2. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***PowerShell***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Remarque** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *Bash*, basculez-le vers ***PowerShell***.

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

Nous allons maintenant créer un notebook Spark et importer les données avec lesquelles nous allons travailler dans cet exercice.

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.

1. Remplacez le nom de notebook par défaut (**Notebook sans titre *[date]***) par `Explore Delta Lake`, puis dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

    ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/products.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

1. Sous la cellule de code existante, utilisez l’icône **+ Code** pour ajouter une nouvelle cellule de code. Ensuite, dans la nouvelle cellule, entrez et exécutez le code suivant pour charger les données à partir du fichier et afficher les 10 premières lignes.

    ```python
   df = spark.read.load('/delta_lab/products.csv', format='csv', header=True)
   display(df.limit(10))
    ```

## Charger les données du fichier dans une table delta

Les données ont été chargées dans un dataframe. Nous allons les conserver dans une table delta.

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter le code suivant :

    ```python
   delta_table_path = "/delta/products-delta"
   df.write.format("delta").save(delta_table_path)
    ```

    Les données d’une table delta lake sont stockées au format Parquet. Un fichier journal est également créé pour suivre les modifications apportées aux données.

1. Ajoutez une nouvelle cellule de code et utilisez-la pour exécuter les commandes shell suivantes afin de visualiser le contenu du dossier où les données Delta ont été enregistrées.

    ```
    %sh
    ls /dbfs/delta/products-delta
    ```

1. Les données de fichier au format Delta peuvent être chargées dans un objet **DeltaTable**, que vous pouvez utiliser pour afficher et mettre à jour les données dans la table. Exécutez le code suivant dans une nouvelle cellule pour mettre à jour les données ; réduisez le prix du produit 771 de 10 %.

    ```python
   from delta.tables import *
   from pyspark.sql.functions import *
   
   # Create a deltaTable object
   deltaTable = DeltaTable.forPath(spark, delta_table_path)
   # Update the table (reduce price of product 771 by 10%)
   deltaTable.update(
       condition = "ProductID == 771",
       set = { "ListPrice": "ListPrice * 0.9" })
   # View the updated data as a dataframe
   deltaTable.toDF().show(10)
    ```

    La mise à jour est conservée dans les données du dossier delta et sera reflétée dans n’importe quel nouveau dataframe chargé à partir de cet emplacement.

1. Exécutez le code suivant pour créer un dataframe à partir des données de table delta :

    ```python
   new_df = spark.read.format("delta").load(delta_table_path)
   new_df.show(10)
    ```

## Explorer la journalisation et le *voyage dans le temps*

Les modifications de données sont journalisées, ce qui vous permet d’utiliser les fonctionnalités de *voyage dans le temps* de Delta Lake pour afficher les versions précédentes des données. 

1. Dans une nouvelle cellule de code, utilisez le code suivant pour afficher la version d’origine des données sur les produits :

    ```python
   new_df = spark.read.format("delta").option("versionAsOf", 0).load(delta_table_path)
   new_df.show(10)
    ```

1. Le journal contient un historique complet des modifications apportées aux données. Utilisez le code suivant pour afficher un enregistrement des 10 dernières modifications :

    ```python
   deltaTable.history(10).show(10, False, True)
    ```

## Créer des tables de catalogue

Jusqu’à présent, vous avez travaillé avec des tables delta en chargeant les données du dossier contenant les fichiers Parquet sur lesquels la table est basée. Vous pouvez définir des *tables de catalogue* qui encapsulent les données et fournissent une entité de table nommée que vous pouvez référencer dans le code SQL. Spark prend en charge deux types de tables de catalogue pour delta lake :

- Les tables *externes* définies par le chemin d’accès aux fichiers contenant les données de la table.
- Tables *managées* définies dans le metastore.

### Créer une table externe

1. Utilisez le code suivant pour créer une base de données nommée **AdventureWorks**, puis créez une table externe nommée **ProductsExternal** dans cette base de données en fonction du chemin des fichiers Delta que vous avez définis précédemment :

    ```python
   spark.sql("CREATE DATABASE AdventureWorks")
   spark.sql("CREATE TABLE AdventureWorks.ProductsExternal USING DELTA LOCATION '{0}'".format(delta_table_path))
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsExternal").show(truncate=False)
    ```

    Notez que la propriété **Location** de la nouvelle table est le chemin d’accès que vous avez spécifié.

1. Utilisez le code suivant pour interroger la table :

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsExternal;
    ```

### Créer une table managée

1. Exécutez le code suivant pour créer (puis décrire) une table gérée nommée **ProductsManaged** basée sur le dataframe que vous avez initialement chargé à partir du fichier **products.csv** (avant de mettre à jour le prix du produit 771).

    ```python
   df.write.format("delta").saveAsTable("AdventureWorks.ProductsManaged")
   spark.sql("DESCRIBE EXTENDED AdventureWorks.ProductsManaged").show(truncate=False)
    ```

    Vous n’avez spécifié aucun chemin d’accès pour les fichiers Parquet utilisés par la table. Cela est géré pour vous dans le metastore Hive et affiché dans la propriété **Location** de la description de table.

1. Utilisez le code suivant pour interroger la table managée, en notant que la syntaxe est identique à celle d’une table managée :

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM ProductsManaged;
    ```

### Comparer des tables externes et managées

1. Utilisez le code suivant pour répertorier les tables de la base de données **AdventureWorks** :

    ```sql
   %sql
   USE AdventureWorks;
   SHOW TABLES;
    ```

1. Utilisez maintenant le code suivant pour afficher les dossiers sur lesquels ces tables sont basées :

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

1. Utilisez le code suivant pour supprimer les deux tables de la base de données :

    ```sql
   %sql
   USE AdventureWorks;
   DROP TABLE IF EXISTS ProductsExternal;
   DROP TABLE IF EXISTS ProductsManaged;
   SHOW TABLES;
    ```

1. Réexécutez maintenant la cellule contenant le code suivant pour afficher le contenu des dossiers delta :

    ```Bash
    %sh
    echo "External table:"
    ls /dbfs/delta/products-delta
    echo
    echo "Managed table:"
    ls /dbfs/user/hive/warehouse/adventureworks.db/productsmanaged
    ```

    Les fichiers de la table managée sont supprimés automatiquement lorsque la table est supprimée. Toutefois, les fichiers de la table externe restent. La suppression d’une table externe supprime uniquement les métadonnées de la table de la base de données ; elle ne supprime pas les fichiers de données.

1. Utilisez le code suivant pour créer une table dans la base de données basée sur les fichiers delta dans le dossier **products-delta** :

    ```sql
   %sql
   USE AdventureWorks;
   CREATE TABLE Products
   USING DELTA
   LOCATION '/delta/products-delta';
    ```

1. Utilisez le code suivant pour interroger la nouvelle table :

    ```sql
   %sql
   USE AdventureWorks;
   SELECT * FROM Products;
    ```

    Étant donné que la table est basée sur les fichiers delta existants, qui incluent l’historique journalisé des modifications, elle reflète les modifications que vous avez précédemment apportées aux données sur les produits.

## Optimiser la disposition des tables

Le stockage physique des données de table et des données d’index associées peut être réorganisé afin de réduire l’espace de stockage et d’améliorer l’efficacité des E/S lors de l’accès à la table. Cela est particulièrement utile après des opérations importantes d’insertion, de mise à jour ou de suppression sur une table.

1. Dans une nouvelle cellule de code, utilisez le code suivant pour optimiser la disposition et nettoyer les anciennes versions des fichiers de données dans la table Delta :

     ```python
    spark.sql("OPTIMIZE Products")
    spark.conf.set("spark.databricks.delta.retentionDurationCheck.enabled", "false")
    spark.sql("VACUUM Products RETAIN 24 HOURS")
     ```

Delta Lake effectue un contrôle de sécurité pour vous éviter d’exécuter une commande VACUUM dangereuse. Dans Databricks Runtime, si vous êtes certain qu’aucune opération effectuée sur cette table ne prend plus de temps que l’intervalle de rétention que vous comptez spécifier, vous pouvez désactiver ce contrôle de sécurité en réglant la propriété de configuration Spark `spark.databricks.delta.retentionDurationCheck.enabled` sur `false`.

> **Remarque** : si vous exécutez VACUUM sur une table Delta, vous perdez la possibilité de remonter dans le temps jusqu’à une version antérieure à la période de conservation des données spécifiée.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
