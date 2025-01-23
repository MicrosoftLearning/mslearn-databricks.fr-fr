---
lab:
  title: "Automatiser l’ingestion et le traitement des données à l’aide d’Azure\_Databricks"
---

# Automatiser l’ingestion et le traitement des données à l’aide d’Azure Databricks

Databricks Jobs est un service puissant qui permet l’automatisation des workflows d’ingestion et de traitement des données. Il permet l’orchestration de pipelines de données complexes, qui peuvent inclure des tâches telles que l’ingestion de données brutes à partir de différentes sources, la transformation de ces données à l’aide de Delt Live Tables et leur persistance vers Delta Lake pour une analyse plus approfondie. Avec Azure Databricks, les utilisateurs peuvent planifier et exécuter automatiquement leurs tâches de traitement des données, ce qui garantit que les données sont toujours à jour et disponibles pour les processus décisionnels.

Ce labo prend environ **20** minutes.

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

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous patientez, consultez l’article [Planifier et orchestrer des workflows](https://learn.microsoft.com/azure/databricks/jobs/) dans la documentation d’Azure Databricks.

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

## Créer un notebook et obtenir des données sources

1. Dans la barre latérale, utilisez le lien **(+) Nouveau** pour créer un **notebook** et modifiez le nom du notebook par défaut (**Notebook sans titre *[date]***) en **Traitement de données**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

2. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales_data.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_data.csv
     ```

3. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.

## Automatiser le traitement de données avec les travaux Azure Databricks

1. Remplacez le code au début du notebook par le code suivant. Ensuite, exécutez-le pour charger les données dans un dataframe :

     ```python
    # Load the sample dataset into a DataFrame
    df = spark.read.csv('/FileStore/*.csv', header=True, inferSchema=True)
    df.show()
     ```

1. Déplacez la souris sous la cellule de code existante, puis utilisez l’icône **+ Code** qui apparaît pour ajouter une nouvelle cellule de code. Dans la nouvelle cellule, entrez et exécutez le code suivant pour agréger les données de ventes par catégorie de produit :

     ```python
    from pyspark.sql.functions import col, sum

    # Aggregate sales data by product category
    sales_by_category = df.groupBy('product_category').agg(sum('transaction_amount').alias('total_sales'))
    sales_by_category.show()
     ```

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **travail**.

1. Remplacez le nom du travail par défaut (**Nouveau travail *[date]***) par `Automated job`.

1. Configurez la tâche sans nom dans le travail avec les paramètres suivants :
    - **Nom de la tâche** : `Run notebook`
    - **Type** : notebook
    - **Source** : espace de travail
    - **Chemin d’accès** : *sélectionnez votre* *notebook* Traitement de données
    - **Cluster** : *Sélectionner votre cluster*

1. Sélectionnez **Créer une tâche**.

1. Sélectionnez **Exécuter maintenant**.

    **Conseil** : dans le volet droit, sous **Planification**, vous pouvez sélectionner **Ajouter un déclencheur** et configurer une planification pour l’exécution du travail (par exemple, quotidienne, hebdomadaire). Toutefois, pour cet exercice, nous allons l’exécuter manuellement.

1. Sélectionnez l’onglet **Exécutions** dans le panneau Travail et surveillez l’exécution du travail.

1. Une fois l’exécution du travail réussie, vous pouvez le sélectionner dans la liste **Exécutions** et vérifier sa sortie.

    Vous avez correctement configuré et automatisé l’ingestion et le traitement des données à l’aide d’Azure Databricks Jobs. Vous pouvez désormais mettre à l’échelle cette solution pour gérer des pipelines de données plus complexes et les intégrer à d’autres services Azure pour une architecture robuste de traitement des données.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
