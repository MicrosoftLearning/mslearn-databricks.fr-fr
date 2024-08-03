---
lab:
  title: "Implémentation de la confidentialité et de la gouvernance des données à l’aide de Microsoft\_Purview et Unity\_Catalog avec Azure\_Databricks"
---

# Implémentation de la confidentialité et de la gouvernance des données à l’aide de Microsoft Purview et Unity Catalog avec Azure Databricks

Microsoft Purview permet une gouvernance complète des données dans l’ensemble de votre paysage de données, en s’intégrant en toute transparence à Azure Databricks pour gérer les données Lakehouse et intégrer les métadonnées dans le mappage de données. Unity Catalog améliore cela en fournissant une gestion et une gouvernance centralisées des données, ce qui simplifie la sécurité et la conformité entre les espaces de travail Databricks.

Ce labo prend environ **30** minutes.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.

2. Utilisez le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, en sélectionnant un environnement ***PowerShell*** et en créant le stockage si vous y êtes invité. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Remarque** : si vous avez créé un shell cloud qui utilise un environnement *Bash*, utilisez le menu déroulant en haut à gauche du volet Cloud Shell pour le remplacer par ***PowerShell***.

3. Notez que vous pouvez redimensionner le volet Cloud Shell en faisant glisser la barre de séparation en haut du volet. Vous pouvez aussi utiliser les icônes **&#8212;** , **&#9723;** et **X** situées en haut à droite du volet pour réduire, agrandir et fermer le volet. Pour plus d’informations sur l’utilisation d’Azure Cloud Shell, consultez la [documentation Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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

9. Dans l’explorateur de catalogues, accédez au volume `sample_data` et vérifiez que les nouvelles tables sont à l’intérieur.
    
## Configurer Microsoft Purview

Microsoft Purview est un service de gouvernance des données unifié qui aide les organisations à gérer et sécuriser leurs données dans différents environnements. Avec des fonctionnalités telles que la protection contre la perte de données, la protection des informations et la gestion de la conformité, Microsoft Purview fournit des outils pour comprendre, gérer et protéger les données tout au long de son cycle de vie.

1. Accédez au [portail Azure](https://portal.azure.com/).

2. Sélectionnez **Créer une ressource** et recherchez **Microsoft Purview**.

3. Créez une ressource **Microsoft Purview** avec les paramètres suivants :
    - **Abonnement** : *Sélectionnez votre abonnement Azure*.
    - **Groupe de ressources** : *choisissez le même groupe de ressources que votre espace de travail Azure Databricks*
    - **Nom du compte Microsoft Purview** : *nom unique de votre choix*
    - **Emplacement** : *sélectionnez la même région que votre espace de travail Azure Databricks*

4. Sélectionnez **Vérifier + créer**. Attendez la fin de la validation, puis sélectionnez **Créer**.

5. Attendez la fin du déploiement. Accédez ensuite à la ressource Microsoft Purview déployée dans le portail Azure.

6. Dans le portail de gouvernance Microsoft Purview, accédez à la section **Mappage de données** dans la barre latérale.

7. Dans le volet **Sources de données**, sélectionnez **Enregistrer**.

8. Dans la fenêtre **Enregistrer la source de données**, recherchez et sélectionnez **Azure Databricks**. Sélectionnez **Continuer**.

9. Donnez à votre source de données un nom unique, puis sélectionnez votre espace de travail Azure Databricks. Sélectionnez **Inscrire**.

## Implémenter des stratégies de confidentialité et de gouvernance des données

1. Dans la section **Mappage de données** de la barre latérale, sélectionnez **Classifications**.

2. Dans le volet **Classifications**, sélectionnez **+ Nouveau** et créez une classification nommée **Informations d’identification personnelle**. Cliquez sur **OK**.

3. Sélectionnez **Data Catalog** dans la barre latérale et accédez à la table **clients**.

4. Appliquez la classification des informations d’identification personnelle aux colonnes de messagerie et de téléphone.

5. Accédez à Azure Databricks et ouvrez le notebook créé précédemment.
 
6. Dans une nouvelle cellule, exécutez le code suivant pour créer une stratégie d’accès aux données de sorte à restreindre l’accès aux informations d’identification personnelle.

     ```sql
    CREATE OR REPLACE TABLE ecommerce.customers (
      customer_id STRING,
      name STRING,
      email STRING,
      phone STRING,
      address STRING,
      city STRING,
      state STRING,
      zip_code STRING,
      country STRING
    ) TBLPROPERTIES ('data_classification'='PII');

    GRANT SELECT ON TABLE ecommerce.customers TO ROLE data_scientist;
    REVOKE SELECT (email, phone) ON TABLE ecommerce.customers FROM ROLE data_scientist;
     ```

7. Essayez d’interroger la table clients en tant qu’utilisateur avec le rôle data_scientist. Vérifiez que l’accès aux colonnes Informations d'identification personnelle (e-mail et téléphone) est restreint.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
