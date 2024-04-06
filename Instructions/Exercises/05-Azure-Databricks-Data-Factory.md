---
lab:
  title: "Automatiser un notebook Azure\_Databricks avec Azure\_Data\_Factory"
---

# Automatiser un notebook Azure Databricks avec Azure Data Factory

Vous pouvez utiliser des notebooks dans Azure Databricks pour effectuer des tâches d’ingénierie des données, telles que le traitement des fichiers de données et le chargement de données dans des tables. Lorsque vous devez orchestrer ces tâches dans le cadre d’un pipeline d’ingénierie des données, vous pouvez utiliser Azure Data Factory.

Cet exercice devrait prendre environ **40** minutes.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota ou d’autorisations insuffisant, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez [Présentation d’Azure Data Factory](https://docs.microsoft.com/azure/data-factory/introduction).

## Créer une ressource Azure Data Factory

En plus de votre espace de travail Azure Databricks, vous devrez approvisionner votre abonnement avec une ressource Azure Data Factory.

1. Dans le Portail Azure, fermez le volet Cloud Shell et accédez au groupe de ressources ***msl-*xxxxxxx*** créé par le script d’installation (ou au groupe de ressources qui contient votre espace de travail Azure Databricks existant).
1. Dans la barre d’outils, sélectionnez **+ Créer** et recherchez `Data Factory`. Créez ensuite une ressource **Data Factory** avec les paramètres suivants :
    - **Abonnement** : *votre abonnement*.
    - **Groupe de ressources** : msl-*xxxxxxx* (ou le groupe de ressources qui contient votre espace de travail Azure Databricks existant)
    - **Nom** : *Un nom unique, par exemple **adf-xxxxxxx***
    - **Région** : *La même région que votre espace de travail Azure Databricks (ou toute autre région disponible si celle-ci n’est pas répertoriée)*
    - **Version** : V2
1. Lorsque la nouvelle ressource est créée, vérifiez que le groupe de ressources contient à la fois l’espace de travail Azure Databricks et les ressources Azure Data Factory.

## Créer un notebook

Vous pouvez créer des notebooks dans votre espace de travail Azure Databricks pour exécuter du code écrit dans divers langages de programmation. Dans cet exercice, vous allez créer un notebook simple qui ingère des données à partir d’un fichier et les enregistre dans un dossier du système de fichiers Databricks (DBFS).

1. Dans le Portail Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou accédez au groupe de ressources qui contient votre espace de travail Azure Databricks existant)
1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Affichez le portail de l’espace de travail Azure Databricks et notez que la barre latérale gauche contient des icônes indiquant les différentes tâches que vous pouvez effectuer.
1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
1. Modifier le nom du notebook par défaut (**Notebook sans titre *[date]***) en **Traiter les données**.
1. Dans la première cellule du notebook, saisissez (mais n’exécutez pas) le code suivant pour définir une variable pour le dossier dans lequel ce notebook enregistrera des données.

    ```python
   # Use dbutils.widget define a "folder" variable with a default value
   dbutils.widgets.text("folder", "data")
   
   # Now get the parameter value (if no value was passed, the default set above will be used)
   folder = dbutils.widgets.get("folder")
    ```

1. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Saisissez ensuite dans la nouvelle cellule (sans l’exécuter) le code suivant pour télécharger les données et les enregistrer dans le dossier :

    ```python
   import urllib3
   
   # Download product data from GitHub
   response = urllib3.PoolManager().request('GET', 'https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv')
   data = response.data.decode("utf-8")
   
   # Save the product data to the specified folder
   path = "dbfs:/{0}/products.csv".format(folder)
   dbutils.fs.put(path, data, True)
    ```

1. Dans la barre latérale de gauche, sélectionnez **Espace de travail** et vérifiez que vos notebooks **Traiter les données** sont répertoriés. Vous allez utiliser Azure Data Factory pour exécuter le notebook dans le cadre d’un pipeline.

    > **Remarque** : Le notebook peut contenir pratiquement toute logique de traitement des données dont vous avez besoin. Cet exemple simple est conçu pour illustrer les principes fondamentaux.

## Activer l’intégration d’Azure Databricks avec Azure Data Factory

Pour utiliser Azure Databricks à partir d’un pipeline Azure Data Factory, vous devez créer un service lié dans Azure Data Factory qui permet d’accéder à votre espace de travail Azure Databricks.

### Générer un jeton d’accès

1. Dans le portail Azure Databricks, dans la barre de menus en haut à droite, sélectionnez le nom d’utilisateur, puis **Paramètres utilisateurs** dans la liste déroulante.
1. Dans la page **Paramètres utilisateurs**, sélectionnez **Développeur**. Ensuite, en regard de **Jetons d’accès**, sélectionnez **Gérer**.
1. Sélectionnez **Générer un nouveau jeton** et générez un nouveau jeton avec le commentaire *Data Factory* et une durée de vie vide (le jeton n’expire donc pas). Veillez à **copier le jeton lorsqu’il est affiché <u>avant</u> de sélectionner *Terminé***.
1. Collez le jeton copié dans un fichier texte afin de pouvoir l’utiliser ultérieurement dans cet exercice.

### Créer un service lié dans Azure Data Factory

1. Revenez au portail Azure et, dans le groupe de ressources **msl-*xxxxxxx***, sélectionnez la ressource Azure Data Factory **adf*xxxxxxx***.
2. Dans la page **Vue d’ensemble**, sélectionnez **Lancer Studio** pour ouvrir Azure Data Factory Studio. Connectez-vous si vous y êtes invité.
3. Dans Azure Data Factory Studio, utilisez l’icône **>>** pour développer le volet de navigation à gauche. Sélectionnez ensuite la page **Gérer**.
4. Dans la page **Gérer**, sous l’onglet **Services liés**, sélectionnez **+ Nouveau** pour ajouter un nouveau service lié.
5. Dans le volet **Nouveau service lié**, sélectionnez l’onglet **Calcul** dans la partie supérieure. Sélectionnez ensuite **Azure Databricks**.
6. Créez ensuite le service lié avec les paramètres suivants :
    - **Nom** : AzureDatabricks
    - **Description** : espace de travail Azure Databricks
    - **Se connecter via un runtime d’intégration** : AutoResolveIntegrationRuntime
    - **Méthode de sélection du compte** : à partir d’un abonnement Azure
    - **Abonnement Azure** : *sélectionnez votre abonnement*
    - **Espace de travail Databricks** : *sélectionnez votre espace de travail **databricksxxxxxxx***
    - **Sélectionner un cluster** : nouveau cluster de travail
    - **URL de l’espace de travail Databricks** : *définie automatiquement sur l’URL de votre espace de travail Databricks*
    - **Type d’authentification** : jeton d’accès
    - **Jeton d’accès** : *collez votre jeton d’accès*
    - **Version du cluster** : 13.3 LTS (Spark 3.4.1, Scala 2.12)
    - **Type de nœud de cluster** : Standard_DS3_v2
    - **Version de Python** : 3
    - **Options de Worker** : fixe
    - **Workers** : 1

## Utiliser un pipeline pour exécuter le notebook Azure Databricks

Maintenant que vous avez créé un service lié, vous pouvez l’utiliser dans un pipeline pour exécuter le notebook que vous avez consulté précédemment.

### Créer un pipeline

1. Dans Azure Data Factory Studio, sélectionnez **Créer** dans le volet de navigation.
2. Dans la page **Créer**, dans le volet **Ressources Factory**, utilisez l’icône **+** pour ajouter un **pipeline**.
3. Dans le volet **Propriétés** du nouveau pipeline, remplacez son nom par **Traiter les données avec Databricks**. Utilisez ensuite le bouton **Propriétés** (qui ressemble à **<sub>*</sub>**) à droite de la barre d’outils pour masquer le volet **Propriétés**.
4. Dans le volet **Activités**, développez **Databricks**, puis faites glisser une activité **Notebook** vers la surface du concepteur de pipeline.
5. Une fois la nouvelle activité **Notebook1** sélectionnée, définissez les propriétés suivantes dans le volet du bas :
    - **Général :**
        - **Nom** : traiter les données
    - **Azure Databricks** :
        - **Service lié Databricks** : *sélectionnez le service lié **AzureDatabricks** que vous avez créé précédemment*
    - **Paramètres**:
        - **Chemin d’accès au notebook** : *Naviguez jusqu’au dossier **Utilisateurs/votre_nom_d’utilisateur** et sélectionnez le notebook **Traiter les données***
        - **Paramètres de base**: *Ajouter un nouveau paramètre nommé `folder` avec la valeur `product_data`*
6. Utilisez le bouton **Valider** au-dessus de la surface du concepteur de pipeline pour valider le pipeline. Utilisez ensuite le bouton **Publier tout** pour le publier (l’enregistrer).

### Exécuter le pipeline

1. Au-dessus de la surface du concepteur de pipeline, sélectionnez **Ajouter un déclencheur**, puis sélectionnez **Déclencher maintenant**.
2. Dans le volet **Exécution de pipeline**, sélectionnez **OK** pour exécuter le pipeline.
3. Dans le volet de navigation de gauche, sélectionnez **Superviser** et observez le pipeline **Traiter les données avec Databricks** sous l’onglet **Exécutions de pipeline**. L’exécution peut prendre un certain temps en raison de la création dynamique d’un cluster Spark et de l’exécution du notebook. Vous pouvez utiliser le bouton **↻ Actualiser** sur la page **Exécutions de pipeline** pour actualiser l’état.

    > **Remarque** : si votre pipeline échoue, il se peut que le quota de votre abonnement soit insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné pour créer un cluster de travail. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./setup.ps1 eastus`

4. Une fois l’exécution réussie, sélectionnez son nom pour afficher les détails de l’exécution. Ensuite, dans la page **Traiter les données avec Databricks**, dans la section **Exécutions d’activité**, sélectionnez l’activité **Traiter les données** et utilisez son icône de ***sortie*** pour afficher le code JSON de sortie de l’activité, qui doit ressembler à ceci :

    ```json
    {
        "runPageUrl": "https://adb-..../run/...",
        "runOutput": "dbfs:/product_data/products.csv",
        "effectiveIntegrationRuntime": "AutoResolveIntegrationRuntime (East US)",
        "executionDuration": 61,
        "durationInQueue": {
            "integrationRuntimeQueue": 0
        },
        "billingReference": {
            "activityType": "ExternalActivity",
            "billableDuration": [
                {
                    "meterType": "AzureIR",
                    "duration": 0.03333333333333333,
                    "unit": "Hours"
                }
            ]
        }
    }
    ```

5. Notez la valeur **runOutput**, qui est la variable de *chemin d’accès* à laquelle le notebook a enregistré les données.

## Nettoyage

Si vous avez terminé l’exploration d’Azure Databricks, vous pouvez supprimer les ressources que vous avez créées afin d’éviter des coûts Azure non nécessaires et de libérer de la capacité dans votre abonnement.
