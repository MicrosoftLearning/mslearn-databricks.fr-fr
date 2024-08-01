---
lab:
  title: "Implémenter des pipelines CI/CD avec Azure Databricks et Azure\_DevOps ou Azure\_Databricks et GitHub"
---

# Implémenter des pipelines CI/CD avec Azure Databricks et Azure DevOps ou Azure Databricks et GitHub

L’implémentation de pipelines d’intégration continue (CI) et de déploiement continu (CD) avec Azure Databricks et Azure DevOps ou Azure Databricks et GitHub implique la configuration d’une série d’étapes automatisées pour garantir que les modifications de code soient intégrées, testées et déployées efficacement. Le processus inclut généralement la connexion à un référentiel Git, l’exécution de travaux à l’aide d’Azure Pipelines pour générer et effectuer le test unitaire du code, ou encore le déploiement des artefacts de build à utiliser dans les notebooks Databricks. Ce workflow permet un cycle de développement robuste, pour une intégration et une livraison continues qui s’alignent sur les pratiques DevOps modernes.

Ce labo prend environ **40** minutes.

>**Remarque :** vous avez besoin d’un compte Github et d’un accès Azure DevOps pour effectuer cet exercice.

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

## Créer un notebook et ingérer des données

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

2. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

3. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.
   
## Configurer un référentiel GitHub et un projet Azure DevOps

Une fois que vous avez connecté un référentiel GitHub à un projet Azure DevOps, vous pouvez configurer des pipelines CI qui se déclenchent avec les modifications apportées à votre référentiel.

1. Accédez à votre [compte GitHub](https://github.com/) et créez un référentiel pour votre projet.

2. Clonez le référentiel sur votre ordinateur local à l’aide de `git clone`.

3. Enregistrez le [fichier CSV](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv) sur votre référentiel local et validez vos modifications.

4. Téléchargez le [notebook Databricks](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc) qui sera utilisé pour lire le fichier CSV et effectuer la transformation des données. Validez les modifications :

5. Accédez au [portail Azure DevOps](https://azure.microsoft.com/en-us/products/devops/) et créez un projet.

6. Dans votre projet Azure DevOps, accédez à la section **Référentiel** et sélectionnez **Importer** pour la connecter à votre référentiel GitHub.

7. Dans la barre de gauche, accédez aux **paramètres du projet > Connexions de service**.

8. Sélectionnez **Créer une connexion de service**, puis **Azure Resource Manager**.

9. Dans **Méthode d’authentification**, sélectionnez **Fédération des identités de charge de travail (automatique)**. Cliquez sur **Suivant**.

10. Dans **Niveau d’étendue**, sélectionnez **Abonnement**. Sélectionnez l’abonnement et le groupe de ressources où vous avez créé votre espace de travail Databricks.

11. Saisissez un nom pour votre connexion de service, puis cochez la case **Accorder une autorisation d’accès à tous les pipelines**. Cliquez sur **Enregistrer**.

Votre projet DevOps a maintenant accès à votre espace de travail Databricks et vous pouvez le connecter à vos pipelines.

## Configurer un pipeline CI

1. Dans la barre de gauche, accédez à **Pipelines** et sélectionnez **Créer un pipeline**.

2. Sélectionnez **GitHub** comme source et sélectionnez votre référentiel.

3. Dans le volet **Configurer votre pipeline**, sélectionnez **Pipeline de démarrage** et utilisez la configuration YAML suivante pour le pipeline CI :

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.x'
    addToPath: true

- script: |
    pip install databricks-cli
  displayName: 'Install Databricks CLI'

- script: |
    databricks fs cp dbfs:/FileStore/sample_sales.csv .
  displayName: 'Download Sample Data from DBFS'

- script: |
    python -m unittest discover -s tests
  displayName: 'Run Unit Tests'
```

4. Sélectionnez **Enregistrer et exécuter**.

Ce fichier YAML configure un pipeline CI déclenché par des modifications apportées à la branche `main` de votre référentiel. Le pipeline configure un environnement Python, installe l’interface CLI Databricks, télécharge les exemples de données à partir de votre espace de travail Databricks et exécute des tests unitaires Python. Il s’agit d’une configuration courante pour les workflows CI.

## Configurer un pipeline CD

1. Dans la barre de gauche, accédez à **Pipelines** > Versions, puis sélectionnez **Créer une version**.

2. Sélectionnez votre pipeline de build comme source d’artefact.

3. Ajoutez une étape et configurez les tâches à déployer sur Azure Databricks :

```yaml
stages:
- stage: Deploy
  jobs:
  - job: DeployToDatabricks
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true

    - script: |
        pip install databricks-cli
      displayName: 'Install Databricks CLI'

    - script: |
        databricks workspace import_dir /path/to/notebooks /Workspace/Notebooks
      displayName: 'Deploy Notebooks to Databricks'
```

Avant d’exécuter ce pipeline, remplacez `/path/to/notebooks` par le chemin d’accès au répertoire où vous disposez de votre notebook dans votre référentiel et `/Workspace/Notebooks` par le chemin d’accès au fichier où vous souhaitez enregistrer le notebook dans votre espace de travail Databricks.

4. Sélectionnez **Enregistrer et exécuter**.

## Exécuter les pipelines

1. Dans votre référentiel local, ajoutez la ligne suivante à la fin du fichier `sample_sales.csv` :

     ```sql
    2024-01-01,ProductG,1,500
     ```

2. Validez et envoyez (push) vos modifications au référentiel GitHub distant.

3. Les modifications apportées au référentiel déclenchent le pipeline CI. Vérifiez que l’exécution du pipeline CI se termine avec succès.

4. Créez une version dans le pipeline de mise en production et déployez les notebooks sur Databricks. Vérifiez que les notebooks sont déployés et exécutés correctement dans votre espace de travail Databricks.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.







