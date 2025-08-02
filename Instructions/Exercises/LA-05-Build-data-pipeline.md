---
lab:
  title: Créer un pipeline de données avec Delta Live Tables
---

# Créer un pipeline de données avec Delta Live Tables

Delta Live Tables est un framework déclaratif permettant de créer des pipelines de traitement de données fiables, gérables et testables. Un pipeline est l’unité principale utilisée pour configurer et exécuter des workflows de traitement des données avec Delta Live Tables. Il lie des sources de données à des jeux de données cibles via un graphe orienté acyclique (DAG) déclaré en Python ou SQL.

Ce labo prend environ **40** minutes.

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

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Qu’est-ce que Delta Live Tables ?](https://learn.microsoft.com/azure/databricks/delta-live-tables/) dans la documentation d’Azure Databricks.

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

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.

2. Remplacez le nom de notebook par défaut (**Notebook sans titre *[date]***) par `Create a pipeline with Delta Live tables`, puis dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

3. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/delta_lab
    mkdir /dbfs/delta_lab
    wget -O /dbfs/delta_lab/covid_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/covid_data.csv
     ```

4. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

## Créer un pipeline Delta Live Tables à l’aide de SQL

1. Créez un notebook et renommez-le `Pipeline Notebook`.

1. En regard du nom du notebook, sélectionnez **Python** et remplacez le langage par défaut par **SQL**.

1. Entrez le code suivant dans la première cellule sans l’exécuter. Toutes les cellules seront exécutées une fois le pipeline créé. Ce code définit une table Delta Live Table qui sera remplie par les données brutes précédemment téléchargées :

     ```sql
    CREATE OR REFRESH LIVE TABLE raw_covid_data
    COMMENT "COVID sample dataset. This data was ingested from the COVID-19 Data Repository by the Center for Systems Science and Engineering (CSSE) at Johns Hopkins University."
    AS
    SELECT
      Last_Update,
      Country_Region,
      Confirmed,
      Deaths,
      Recovered
    FROM read_files('dbfs:/delta_lab/covid_data.csv', format => 'csv', header => true)
     ```

1. Sous la première cellule, cliquez sur l’icône **+ Code** pour ajouter une nouvelle cellule et entrez le code suivant pour interroger, filtrer et mettre en forme les données de la table précédente avant l’analyse.

     ```sql
    CREATE OR REFRESH LIVE TABLE processed_covid_data(
      CONSTRAINT valid_country_region EXPECT (Country_Region IS NOT NULL) ON VIOLATION FAIL UPDATE
    )
    COMMENT "Formatted and filtered data for analysis."
    AS
    SELECT
        TO_DATE(Last_Update, 'MM/dd/yyyy') as Report_Date,
        Country_Region,
        Confirmed,
        Deaths,
        Recovered
    FROM live.raw_covid_data;
     ```

1. Dans une troisième nouvelle cellule de code, entrez le code suivant, qui créera une vue de données enrichie pour une analyse plus poussée une fois le pipeline exécuté avec succès.

     ```sql
    CREATE OR REFRESH LIVE TABLE aggregated_covid_data
    COMMENT "Aggregated daily data for the US with total counts."
    AS
    SELECT
        Report_Date,
        sum(Confirmed) as Total_Confirmed,
        sum(Deaths) as Total_Deaths,
        sum(Recovered) as Total_Recovered
    FROM live.processed_covid_data
    GROUP BY Report_Date;
     ```
     
1. Sélectionnez **Delta Live Tables** dans la barre latérale gauche, puis sélectionnez **Créer un pipeline**.

1. Dans la page **Créer un pipeline**, créez un pipeline avec les paramètres suivants :
    - **Nom du pipeline** : `Covid Pipeline`
    - **Édition du produit** : Avancé
    - **Mode pipeline** : déclenché
    - **Code source** : *accédez à votre* notebook *Pipeline Notebook dans le**dossier* Users/user@name.
    - **Options de stockage** : metastore Hive
    - **Emplacement de stockage** : `dbfs:/pipelines/delta_lab`
    - **Schéma cible** : *entrez* `default`.

1. Sélectionnez **Créer**, puis **Démarrer**. Attendez ensuite que le pipeline s’exécute (ce qui peut prendre un certain temps).
 
1. Une fois le pipeline correctement exécuté, revenez au récent notebook *Créer un pipeline avec Delta Live Tables* que vous avez créé en premier, puis exécutez le code suivant dans une nouvelle cellule pour vérifier que les fichiers de toutes les 3 nouvelles tables ont été créés à l’emplacement de stockage spécifié :

     ```python
    display(dbutils.fs.ls("dbfs:/pipelines/delta_lab/schemas/default/tables"))
     ```

1. Ajoutez une autre cellule de code et exécutez le code suivant pour vérifier que les tables ont été créées dans la base de données **par défaut** :

     ```sql
    %sql

    SHOW TABLES
     ```

## Visualiser les résultats

Après avoir créé les tables, il est possible de les charger dans des dataframes et de visualiser les données.

1. Dans le notebook *Créer un pipeline avec Delta Live Tables*, ajoutez une nouvelle cellule de code et exécutez le code suivant pour charger `aggregated_covid_data` dans un dataframe :

    ```sql
    %sql
    
    SELECT * FROM aggregated_covid_data
    ```

1. Au-dessus du tableau des résultats, sélectionnez **+**, puis **Visualisation** pour afficher l’éditeur de visualisation et appliquer les options suivantes :
    - **Type de visualisation** : ligne
    - **Colonne X** : Report_Date
    - **Colonne Y** : *ajoutez une nouvelle colonne et sélectionnez***Total_Confirmed**. *Appliquez* **l’agrégation** *Sum*.

1. Enregistrez la visualisation, puis affichez le graphique résultant dans le notebook.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
