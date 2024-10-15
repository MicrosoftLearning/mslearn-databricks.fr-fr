---
lab:
  title: "Déconseillé - Explorer Azure\_Databricks"
---

# Explorer Azure Databricks

Azure Databricks est une version basée sur Microsoft Azure de la plateforme Databricks open source populaire.

De même qu’Azure Synapse Analytics, un *espace de travail* Azure Databricks permet la gestion des clusters, des données et des ressources Databricks sur Azure dans un emplacement unique.

Cet exercice devrait prendre environ **30** minutes.

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
7. Attendez que le script se termine. Cette opération prend généralement environ 5 minutes, mais dans certains cas, elle peut être plus longue. Pendant que vous patientez, consultez l’article [Présentation d’Azure Databricks](https://learn.microsoft.com/azure/databricks/introduction/) dans la documentation d’Azure Databricks.

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

## Utiliser Spark pour analyser des données

Comme dans de nombreux environnements Spark, Databricks prend en charge l’utilisation de notebooks pour combiner des notes et des cellules de code interactives que vous pouvez utiliser pour explorer les données.

1. Téléchargez le fichier [**products.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv) à partir de `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv` vers votre ordinateur local, en l’enregistrant en tant que **products.csv**.
1. Dans la barre latérale, dans le menu du lien **(+) Nouveau**, sélectionnez **Téléchargement du fichier**.
1. Téléchargez le fichier **products.csv** téléchargé sur votre ordinateur.
1. Dans la page **Créer ou modifier une table à partir du chargement de fichier**, veillez à sélectionner votre cluster en haut de la page. Choisissez ensuite le catalogue **hive_metastore** et son schéma par défaut pour créer une table nommée **produits**.
1. Dans la page **Explorateur de catalogue** une fois la page **produits** créée, dans le menu du bouton **Créer**, sélectionnez **Notebook** pour créer un notebook.
1. Dans le notebook, vérifiez que le notebook est connecté à votre cluster, puis passez en revue le code automatiquement ajouté dans la première cellule et qui doit ressembler à ce qui suit :

    ```python
    %sql
    SELECT * FROM `hive_metastore`.`default`.`products`;
    ```

1. Utilisez l’option du menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter, en démarrant et en attachant le cluster, si vous y êtes invité.
1. Attendez que l’exécution de la tâche Spark par le code soit terminée. Le code récupère les données de la table créée en fonction du fichier téléchargé.
1. Au-dessus du tableau des résultats, sélectionnez **+**, puis **Visualisation** pour afficher l’éditeur de visualisation et appliquer les options suivantes :
    - **Type de visualisation** : barre
    - **Colonne X** : catégorie
    - **Colonne Y** : *ajoutez une nouvelle colonne et sélectionnez* **ProductID**. *Appliquez l’**agrégation* **Count**.

    Enregistrez la visualisation et observez qu’elle s’affiche dans le notebook comme suit :

    ![Graphique à barres montrant les quantités de produits par catégorie](./images/databricks-chart.png)

## Analyser des données avec un dataframe

Bien que la plupart des analyses de données tolèrent l’utilisation de code SQL tel qu’il est utilisé dans l’exemple précédent, certains analystes des données et scientifiques des données peuvent utiliser des objets Spark natifs de type *dataframe*, dans des langages de programmation tels que *PySpark* (une version de Python optimisée avec Spark), pour utiliser des données de manière efficace.

1. Dans le notebook, sous la sortie de graphique de la cellule de code précédemment exécutée, utilisez l’icône **+** pour ajouter une nouvelle cellule.
1. Entrez et exécutez le code suivant dans la nouvelle cellule :

    ```python
    df = spark.sql("SELECT * FROM products")
    df = df.filter("Category == 'Road Bikes'")
    display(df)
    ```

1. Exécutez la nouvelle cellule qui retourne des produits dans la catégorie *Road Bikes* (Vélos de route).

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
