---
lab:
  title: Explorer Azure Databricks
---

# Explorer Azure Databricks

Azure Databricks est une version basée sur Microsoft Azure de la plateforme Databricks open source populaire.

De même qu’Azure Synapse Analytics, un *espace de travail* Azure Databricks permet la gestion des clusters, des données et des ressources Databricks sur Azure dans un emplacement unique.

Cet exercice devrait prendre environ **30** minutes.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota ou d’autorisations insuffisant, vous pouvez essayer de créer un espace de travail Azure Databricks de manière interactive dans le Portail Microsoft Azure.

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

## Utiliser Spark pour analyser un fichier de données

Comme dans de nombreux environnements Spark, Databricks prend en charge l’utilisation de notebooks pour combiner des notes et des cellules de code interactives que vous pouvez utiliser pour explorer les données.

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
1. Remplacez le nom du notebook par défaut (**Untitled Notebook *[date]***) par **Explorer les produits** et dans la liste déroulante **Connecter**, sélectionnez votre cluster si ce n’est pas déjà le cas. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.
1. Téléchargez le fichier [**products.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv) à partir de `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv` vers votre ordinateur local, en l’enregistrant en tant que **products.csv**. Ensuite, dans le notebook **Explorer les produits**, à partir du menu **Fichier**, sélectionnez **Charger des données dans DBFS**.
1. Dans la boîte de dialogue **Charger des données**, notez le **répertoire cible DBFS** dans lequel le fichier sera chargé. Sélectionnez ensuite la zone **Fichiers**, puis chargez le fichier **products.csv** que vous avez téléchargé sur votre ordinateur. Une fois le fichier chargé, sélectionnez **Suivant**.
1. Dans le volet **Accéder aux fichiers à partir des notebooks**, sélectionnez l’exemple de code PySpark et copiez-le dans le presse-papiers. Vous l’utiliserez pour charger les données à partir du fichier dans un DataFrame. Ensuite, sélectionnez **Terminé**.
1. Dans le notebook **Explorer les produits**, dans la cellule de code vide, collez le code que vous avez copié, qui doit ressembler à ceci :

    ```python
    df1 = spark.read.format("csv").option("header", "true").load("dbfs:/FileStore/shared_uploads/user@outlook.com/products.csv")
    ```

1. Sélectionnez l’option de menu **▸ Exécuter la cellule** en haut à droite de la cellule pour l’exécuter, en démarrant et en attachant le cluster si vous y êtes invité.
1. Attendez que l’exécution de la tâche Spark par le code soit terminée. Le code a créé un objet *dataframe* nommé **df1** à partir des données du fichier que vous avez chargé.
1. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Dans la nouvelle cellule, entrez ensuite le code suivant :

    ```python
   display(df1)
    ```

1. Utilisez l’option de menu **▸ Exécuter la cellule** en haut à droite de la nouvelle cellule pour l’exécuter. Ce code affiche le contenu du dataframe, qui doit ressembler à ceci :

    | ProductID | ProductName | Catégorie | ListPrice |
    | -- | -- | -- | -- |
    | 771 | Mountain-100 Silver, 38 | VTT | 3399.9900 |
    | 772 | Mountain-100 Silver, 42 | VTT | 3399.9900 |
    | ... | ... | ... | ... |

1. Au-dessus du tableau des résultats, sélectionnez **+**, puis **Visualisation** pour afficher l’éditeur de visualisation et appliquer les options suivantes :
    - **Type de visualisation** : barre
    - **Colonne X** : catégorie
    - **Colonne Y** : *ajoutez une nouvelle colonne et sélectionnez* **ProductID**. *Appliquez l’**agrégation* **Count**.

    Enregistrez la visualisation et observez qu’elle s’affiche dans le notebook comme suit :

    ![Graphique à barres montrant les quantités de produits par catégorie](./images/databricks-chart.png)

## Créer et interroger une table

Bien que de nombreuses analyses de données utilisent des langages tels que Python ou Scala pour traiter les données contenues dans des fichiers, de nombreuses solutions d’analytique données reposent sur des bases de données relationnelles, dans lesquelles les données sont stockées dans des tables et manipulées à l’aide de SQL.

1. Dans le notebook **Explorer les produits**, sous la sortie de graphique de la cellule de code précédemment exécutée, sélectionnez l’icône **+** pour ajouter une nouvelle cellule.
2. Entrez et exécutez le code suivant dans la nouvelle cellule :

    ```python
   df1.write.saveAsTable("products")
    ```

3. Une fois la cellule terminée, ajoutez une nouvelle cellule sous celle-ci avec le code suivant :

    ```sql
   %sql

   SELECT ProductName, ListPrice
   FROM products
   WHERE Category = 'Touring Bikes';
    ```

4. Exécutez la nouvelle cellule, qui contient du code SQL pour retourner le nom et le prix des produits dans la catégorie *Touring Bikes*.
5. Dans la barre latérale, sélectionnez le lien **Catalogue** et vérifiez que la table **produits** a été créée dans le schéma de base de données par défaut (dont le nom est évidemment **default**). Il est possible d’utiliser du code Spark pour créer des schémas de base de données personnalisés et un schéma de tables relationnelles que les analystes de données peuvent utiliser pour explorer les données et générer des rapports analytiques.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
