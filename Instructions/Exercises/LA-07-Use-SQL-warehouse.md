---
lab:
  title: "Utiliser un entrepôt SQL dans Azure\_Databricks"
---

# Utiliser un entrepôt SQL dans Azure Databricks

SQL est un langage standard pour interroger et manipuler des données. De nombreux analystes de données effectuent des analytiques données à l’aide de SQL pour interroger des tables d’une base de données relationnelle. Azure Databricks inclut la fonctionnalité SQL qui s’appuie sur les technologies Spark et Delta Lake pour fournir une couche de base de données relationnelle sur des fichiers dans un lac de données.

Cet exercice devrait prendre environ **30** minutes.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks *Premium* ou *Essai*, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

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
7. Attendez que le script se termine. Cette opération prend généralement environ 5 minutes, mais dans certains cas, elle peut être plus longue. Pendant que vous patientez, consultez l’article [Présentation de l’entrepôt de données sur Azure Databricks](https://learn.microsoft.com/azure/databricks/sql/) dans la documentation d’Azure Databricks.

## Afficher et démarrer un entrepôt SQL

1. Une fois la ressource d’espace de travail Azure Databricks déployée, accédez-y dans le portail Azure.
1. Dans la page **Vue d’ensemble** de votre espace de travail Azure Databricks, sélectionnez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur, puis connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Affichez le portail de l’espace de travail Azure Databricks et notez que la barre latérale de gauche contient les noms des catégories de tâches.
1. Dans la barre latérale, sous **SQL**, sélectionnez **Entrepôts SQL**.
1. Notez que l’espace de travail inclut déjà un entrepôt SQL nommé **Starter Warehouse**.
1. Dans le menu **Actions** (**⁝**) de l’entrepôt SQL, sélectionnez **Modifier**. Définissez ensuite la propriété **Taille du cluster** sur **2X-Small** et enregistrez vos modifications.
1. Sélectionnez le bouton **Démarrer** pour lancer l’entrepôt SQL (ce qui peut prendre une minute ou deux).

> **Remarque** : si votre entrepôt SQL ne démarre pas, il se peut que le quota de votre abonnement soit insuffisant dans la région où votre espace de travail Azure Databricks est configuré. Consultez l’article [Quota de processeurs virtuels Azure requis](https://docs.microsoft.com/azure/databricks/sql/admin/sql-endpoints#required-azure-vcpu-quota) pour plus de détails. Si cela se produit, vous pouvez essayer de demander une augmentation de quota comme indiqué dans le message d’erreur lorsque l’entrepôt ne parvient pas à démarrer. Vous pouvez également essayer de supprimer votre espace de travail et d’en créer un autre dans une région différente. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./setup.ps1 eastus`

## Créer un schéma de base de données

1. Lorsque votre entrepôt SQL est *en cours d’exécution*, sélectionnez **Éditeur SQL** dans la barre latérale.
2. Dans le volet **Navigateur de schémas**, notez que le catalogue *hive_metastore* contient une base de données nommée **default**.
3. Dans le volet **Nouvelle requête**, entrez le code SQL suivant :

    ```sql
   CREATE DATABASE retail_db;
    ```

4. Sélectionnez le bouton **►Exécuter** pour exécuter le code SQL.
5. Lorsque le code a été correctement exécuté, dans le volet **Navigateur de schémas**, sélectionnez le bouton d’actualisation en bas du volet pour actualiser la liste. Développez ensuite **hive_metastore** et **retail_db**, puis vérifiez que la base de données a été créée, mais ne contient aucune table.

Vous pouvez utiliser la base de données **default** pour vos tables, mais lors de la création d’un magasin de données analytiques, il est préférable de créer des bases de données personnalisées contenant des données spécifiques.

## Créer une table

1. Téléchargez le fichier [**products.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv) sur votre ordinateur local, puis enregistrez-le en tant que **products.csv**.
1. Dans le portail de l’espace de travail Azure Databricks, dans la barre latérale, sélectionnez **(+) Nouveau**, puis **Chargement de fichiers** et chargez le fichier **products.csv** que vous avez téléchargé sur votre ordinateur.
1. Dans la page **Charger des données**, sélectionnez le schéma **retail_db** et définissez le nom de la table sur **produits**. Sélectionnez ensuite **Créer une table** dans le coin inférieur gauche de la page.
1. Une fois la table créée, vérifiez ses détails.

La possibilité de créer une table en important des données à partir d’un fichier facilite le remplissage d’une base de données. Vous pouvez également utiliser Spark SQL pour créer des tables à l’aide de code. Les tables elles-mêmes sont des définitions de métadonnées dans le metastore Hive, et les données qu’elles contiennent sont stockées au format Delta dans le stockage DBFS (Databricks File System).

## Créer une requête

1. Dans la barre latérale, sélectionnez **(+) Nouveau**, puis **Requête**.
2. Dans le volet **Navigateur de schéma**, développez **hive_metastore** et **retail_db**, puis vérifiez que la table des **produits** est répertoriée.
3. Dans le volet **Nouvelle requête**, entrez le code SQL suivant :

    ```sql
   SELECT ProductID, ProductName, Category
   FROM retail_db.products; 
    ```

4. Sélectionnez le bouton **►Exécuter** pour exécuter le code SQL.
5. Une fois la requête terminée, consultez la table des résultats.
6. Sélectionnez le bouton **Enregistrer** en haut à droite de l’éditeur de requête pour enregistrer la requête en tant que **Produits et catégories**.

L’enregistrement d’une requête facilite la récupération des mêmes données ultérieurement.

## Création d’un tableau de bord

1. Dans la barre latérale, sélectionnez **(+) Nouveau**, puis **Tableau de bord**.
2. Dans la boîte de dialogue **Nouveau tableau de bord**, entrez le nom **Tableau de bord de vente au détail**, puis sélectionnez **Enregistrer**.
3. Dans le tableau de bord **Tableau de bord de vente au détail**, dans la liste déroulante **Ajouter**, sélectionnez **Visualisation**.
4. Dans la boîte de dialogue **Ajouter un widget de visualisation**, sélectionnez la requête **Produits et catégories**. Sélectionnez ensuite **Créer une visualisation**, définissez le titre sur **Produits par catégorie**, puis sélectionnez **Créer une visualisation**.
5. Dans l’éditeur de visualisation, définissez les propriétés suivantes :
    - **Type de visualisation** : barre
    - **Graphique horizontal** : sélectionné
    - **Colonne Y** : catégorie
    - **Colonnes X** : ID produit : nombre
    - **Regrouper par** : *laissez le champ vide*
    - **Empilement** : désactivé
    - **Normaliser les valeurs en pourcentage** : <u>Non</u> sélectionné
    - **Valeurs NULL et manquantes** : ne pas afficher dans le graphique

6. Enregistrez la visualisation et affichez-la dans le tableau de bord.
7. Sélectionnez **Fin de l’édition** pour afficher le tableau de bord tel que les utilisateurs le verront.

Les tableaux de bord constituent un excellent moyen de partager des tables de données et des visualisations avec des utilisateurs professionnels. Vous pouvez planifier l’actualisation périodique des tableaux de bord et leur envoi par e-mail aux abonnés.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Entrepôts SQL**, sélectionnez votre entrepôt SQL et sélectionnez **&#9632; Arrêtez** pour l’arrêter.

Si vous avez terminé l’exploration d’Azure Databricks, vous pouvez supprimer les ressources que vous avez créées afin d’éviter des coûts Azure non nécessaires et de libérer de la capacité dans votre abonnement.
