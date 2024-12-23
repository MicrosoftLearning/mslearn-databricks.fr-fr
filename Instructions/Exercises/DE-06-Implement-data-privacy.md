---
lab:
  title: "Implémentation de la confidentialité et de la gouvernance des données à l’aide d’Unity\_Catalog avec Azure\_Databricks"
---

# Implémentation de la confidentialité et de la gouvernance des données à l’aide d’Unity Catalog avec Azure Databricks

Unity Catalog propose une solution de gouvernance centralisée pour les données et l’IA, qui simplifie la sécurité et la gouvernance en fournissant un emplacement unique pour administrer et auditer l’accès aux données. Cette solution prend en charge les listes de contrôle d’accès (ACL) affiné et le masquage dynamique des données, qui sont essentiels pour protéger les informations sensibles. 

Ce labo prend environ **30** minutes.

> **Note** : l’interface utilisateur Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

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

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous patientez, consultez l’article [Gouvernance des données avec Unity Catalog](https://learn.microsoft.com/azure/databricks/data-governance/) dans la documentation d’Azure Databricks.

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

## Configurer Unity Catalog

Les metastores Unity Catalog enregistrent des métadonnées sur les objets sécurisables (tels que les tables, les volumes, les emplacements externes et les partages) et les autorisations qui régissent l’accès à ces objets. Chaque metastore expose un espace de noms de trois niveaux (`catalog`.`schema`.`table`) avec lequel les données peuvent être organisées. Vous devez avoir un metastore pour chaque région dans laquelle votre organisation opère. Pour travailler avec Unity Catalog, les utilisateurs doivent se trouver sur un espace de travail attaché à un metastore dans leur région.

1. Dans la barre latérale, sélectionnez **Catalogue**.

2. Dans l’explorateur de catalogues, un catalogue Unity par défaut portant le nom de votre espace de travail (**databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour le créer) doit être présent. Sélectionnez le catalogue, puis, en haut du volet droit, sélectionnez **Créer un schéma**.

3. Nommez le nouveau schéma **ecommerce**, choisissez l’emplacement de stockage créé avec votre espace de travail, puis sélectionnez **Créer**.

4. Sélectionnez votre catalogue et, dans le volet droit, sélectionnez l’onglet **Espaces de travail**. Vérifiez que votre espace de travail dispose d’un accès `Read & Write`.

## Ingérer des exemples de données dans Azure Databricks

1. Téléchargez les fichiers d’exemples de données :
   * [customers.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/customers.csv)
   * [Products.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/products.csv)
   * [sales.csv](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/DE-05/sales.csv)

2. Dans l’espace de travail Azure Databricks, en haut de l’explorateur de catalogues, sélectionnez **+**, puis **Ajouter des données**.

3. Dans la nouvelle fenêtre, sélectionnez **Charger des fichiers sur le volume**.

4. Dans la nouvelle fenêtre, accédez à votre schéma `ecommerce`, développez-le et sélectionnez **Créer un volume**.

5. Nommez le nouveau volume **sample_data** et sélectionnez **Créer**.

6. Sélectionnez le nouveau volume et chargez les fichiers `customers.csv`, `products.csv` et `sales.csv`. Sélectionnez **Charger**.

7. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

8. Dans la première cellule du notebook, entrez le code suivant pour créer des tables à partir des fichiers CSV :

     ```python
    # Load Customer Data
    customers_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/customers.csv")
    customers_df.write.saveAsTable("ecommerce.customers")

    # Load Sales Data
    sales_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/sales.csv")
    sales_df.write.saveAsTable("ecommerce.sales")

    # Load Product Data
    products_df = spark.read.format("csv").option("header", "true").load("/Volumes/databricksxxxxxxx/ecommerce/sample_data/products.csv")
    products_df.write.saveAsTable("ecommerce.products")
     ```

>**Remarque :** dans le chemin d’accès du fichier `.load`, remplacez `databricksxxxxxxx` par le nom de votre catalogue.

9. Dans l’explorateur de catalogues, accédez au schéma `ecommerce` et vérifiez que les nouvelles tables sont à l’intérieur.
    
## Configurer les listes de contrôles d’accès et le masquage dynamique des données

Les listes de contrôle d’accès (ACL) sont un aspect fondamental de la sécurité des données dans Azure Databricks. Elles vous permettent de configurer des autorisations pour différents objets d’un espace de travail. Avec Unity Catalog, vous pouvez centraliser la gouvernance et l’audit de l’accès aux données, qui fournissent un modèle de sécurité affiné essentiel pour la gestion des données et des ressources d’IA. 

1. Dans une nouvelle cellule, exécutez le code suivant pour créer une vue sécurisée de la table `customers` et restreindre l’accès aux informations d’identification personnelle.

     ```sql
    CREATE VIEW ecommerce.customers_secure_view AS
    SELECT 
        customer_id, 
        name, 
        address,
        city,
        state,
        zip_code,
        country, 
        CASE 
            WHEN current_user() = 'admin_user@example.com' THEN email
            ELSE NULL 
        END AS email, 
        CASE 
            WHEN current_user() = 'admin_user@example.com' THEN phone 
            ELSE NULL 
        END AS phone
    FROM ecommerce.customers;
     ```

2. Interrogez la vue sécurisée :

     ```sql
    SELECT * FROM ecommerce.customers_secure_view
     ```

Vérifiez que l’accès aux colonnes Informations d'identification personnelle (e-mail et téléphone) est restreint, car vous n’accédez pas aux données en tant que `admin_user@example.com`.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
