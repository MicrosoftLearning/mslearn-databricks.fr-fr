---
lab:
  title: Implémenter des workflows CI/CD avec Azure Databricks
---

# Implémenter des workflows CI/CD avec Azure Databricks

L’implémentation de workflows CI/CD avec GitHub Actions et Azure Databricks peut simplifier votre processus de développement et améliorer l’automatisation. GitHub Actions fournit une plateforme puissante pour automatiser les workflows logiciels, notamment l’intégration continue (CI) et la livraison continue (CD). Lorsqu’ils sont intégrés à Azure Databricks, ces workflows peuvent exécuter des tâches de données complexes, telles que l’exécution de notebooks ou le déploiement de mises à jour dans des environnements Databricks. Par exemple, vous pouvez utiliser GitHub Actions pour automatiser le déploiement de notebooks Databricks, gérer les chargements du système de fichiers Databricks et configurer l’interface CLI Databricks au sein de vos workflows. Cette intégration permet un cycle de développement plus efficace et résistant aux erreurs, en particulier pour les applications pilotées par les données.

Ce labo prend environ **40** minutes.

> **Note** : l’interface utilisateur Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

> **Note :** vous avez besoin d’un compte GitHub et d’un client Git (tel que l’outil de ligne de commande Git) installé sur votre ordinateur local pour effectuer cet exercice.

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

7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous patientez, consultez l’article [Exécuter un workflow CI/CD avec un lot de ressources Databricks et GitHub Actions](https://learn.microsoft.com/azure/databricks/dev-tools/bundles/ci-cd-bundles) dans la documentation d’Azure Databricks.

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

1. Dans la barre latérale, utilisez le lien **(+) Nouveau** pour créer un **notebook** et modifiez le nom du notebook par défaut (**Notebook sans titre *[date]***) en **Notebook CICD**. Dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes du *shell* pour télécharger des fichiers de données depuis GitHub dans le système de fichiers utilisé par votre cluster.

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.
   
## Configurer un référentiel GitHub

Une fois que vous avez connecté un référentiel GitHub à un espace de travail Databricks, vous pouvez configurer des pipelines CI/CD dans GitHub Actions qui se déclenchent avec les modifications apportées à votre référentiel.

1. Accédez à votre [compte GitHub](https://github.com/) et créez un référentiel privé avec un nom approprié (par exemple, *databricks-cicd-repo*).

1. Clonez le référentiel vide sur votre machine locale à l’aide de la commande [git clone](https://git-scm.com/docs/git-clone).

1. Téléchargez les fichiers requis pour cet exercice dans le dossier local de votre référentiel :
   - [Fichier CSV](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv)
   - [Databricks Notebook](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc)
   - [Fichier de configuration du travail](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/job-config.json)

1. Dans votre clone local du référentiel Git, [ajoutez](https://git-scm.com/docs/git-add) les fichiers. [Validez](https://git-scm.com/docs/git-commit) ensuite les modifications et [transmettez-les](https://git-scm.com/docs/git-push) vers le référentiel.

## Configurer les secrets du référentiel

Les secrets sont des variables que vous créez dans une organisation, un dépôt ou un environnement de dépôt. Les secrets que vous créez peuvent être utilisés dans les workflows GitHub Actions. GitHub Actions peut uniquement lire un secret si vous incluez explicitement le secret dans un flux de travail.

Lorsque les workflows GitHub Actions doivent accéder aux ressources d’Azure Databricks, les informations d’identification d’authentification sont stockées en tant que variables chiffrées à utiliser avec les pipelines CI/CD.

Avant de créer des secrets de référentiel, vous devez générer un jeton d’accès personnel dans Azure Databricks :

1. Dans votre espace de travail Azure Databricks, sélectionnez l’icône *d’utilisateur* dans la barre du haut, puis sélectionnez **Paramètres** dans la liste déroulante.

2. Sur la page **Développeur**, à côté de **Jetons d’accès**, sélectionnez **Gérer**.

3. Sélectionnez **Générer un nouveau jeton**, puis **Générer**.

4. Copiez le jeton affiché et collez-le dans un emplacement auquel vous pourrez faire référence ultérieurement. Ensuite, sélectionnez **Terminé**.

5. Sur la page de votre référentiel GitHub, sélectionnez l’onglet **Paramètres**.

   ![Onglet Paramètres GitHub](./images/github-settings.png)

6. Dans la barre latérale gauche, sélectionnez **Secrets et variables**, puis sélectionnez **Actions**.

7. Sélectionnez **Nouveau secret de référentiel** et ajoutez chacune de ces variables :
   - **Nom :** DATABRICKS_HOST **Secret :** ajoutez l’URL de votre espace de travail Databricks.
   - **Nom :** DATABRICKS_TOKEN **Secret :** ajoutez le jeton d’accès généré précédemment.

## Configurer des pipelines d’intégration continue/de livraison continue

Maintenant que vous avez stocké les variables nécessaires pour accéder à votre espace de travail Azure Databricks à partir de GitHub, vous allez créer des workflows pour automatiser l’ingestion et le traitement des données, qui se déclencheront chaque fois que le référentiel sera mis à jour.

1. Dans la page de votre référentiel, sélectionnez l’onglet **Actions**.

    ![Onglet GitHub Actions](./images/github-actions.png)

2. Sélectionnez **configurer vous-même un workflow** et entrez le code suivant :

     ```yaml
    name: CI Pipeline for Azure Databricks

    on:
      push:
        branches:
          - main
      pull_request:
        branches:
          - main

    jobs:
      deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
          uses: actions/checkout@v3

        - name: Set up Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.x'

        - name: Install Databricks CLI
          run: |
            pip install databricks-cli

        - name: Configure Databricks CLI
          run: |
            databricks configure --token <<EOF
            ${{ secrets.DATABRICKS_HOST }}
            ${{ secrets.DATABRICKS_TOKEN }}
            EOF

        - name: Download Sample Data from DBFS
          run: databricks fs cp dbfs:/FileStore/sample_sales.csv . --overwrite
     ```

    Ce code installe et configure l’interface CLI Databricks et télécharge les exemples de données dans votre référentiel chaque fois qu’une validation est envoyée (push) ou qu’une demande de tirage (pull request) est fusionnée.

3. Nommez le workflow **CI_pipeline.yml** et sélectionnez **Valider les modifications**. Le pipeline s’exécute automatiquement et vous pouvez vérifier son état sous l’onglet **Actions**.

    Une fois le workflow terminé, il est temps d’effectuer les configurations de votre pipeline CD.

4. Accédez à la page de votre espace de travail, sélectionnez **Calcul**, puis sélectionnez votre cluster.

5. Dans la page du cluster, sélectionnez **Plus ...**, puis **Afficher le code JSON**. Copiez l’ID du cluster.

6. Dans votre référentiel, ouvrez **job-config.json** et remplacez *your_cluster_id* par l’ID de cluster que vous venez de copier. Remplacez également */Workspace/Users/votre_nom_d’utilisateur/votre_notebook* par le chemin d’accès dans votre espace de travail où vous souhaitez stocker le notebook utilisé dans le pipeline. Validez les modifications :

    > **Remarque :** si vous accédez à l’onglet **Actions**, vous verrez que le pipeline CI a recommencé à s’exécuter. Étant donné qu’il est censé se déclencher chaque fois qu’une validation est envoyée, la modification de *job-config.json* déploie le pipeline comme prévu.

7. Dans l’onglet **Actions**, créez un workflow nommé **CD_pipeline.yml** et saisissez le code suivant :

     ```yaml
    name: CD Pipeline for Azure Databricks

    on:
      push:
        branches:
          - main

    jobs:
      deploy:
        runs-on: ubuntu-latest

        steps:
        - name: Checkout code
          uses: actions/checkout@v3

        - name: Set up Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.x'

        - name: Install Databricks CLI
          run: pip install databricks-cli

        - name: Configure Databricks CLI
          run: |
            databricks configure --token <<EOF
            ${{ secrets.DATABRICKS_HOST }}
            ${{ secrets.DATABRICKS_TOKEN }}
            EOF
        - name: Upload Notebook to DBFS
          run: databricks fs cp sample_sales_notebook.dbc dbfs:/Workspace/Users/your_username/your_notebook --overwrite
          env:
            DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}

        - name: Run Databricks Job
          run: |
            databricks jobs create --json-file job-config.json
            databricks jobs run-now --job-id $(databricks jobs list | grep 'CD pipeline' | awk '{print $1}')
          env:
            DATABRICKS_TOKEN: ${{ secrets.DATABRICKS_TOKEN }}
     ```

    Avant de valider les modifications, remplacez `/path/to/your/notebook` par le chemin d’accès au fichier de votre notebook dans votre référentiel et `/Workspace/Users/your_username/your_notebook` par le chemin d’accès au fichier où vous souhaitez importer le notebook dans votre espace de travail Azure Databricks.

8. Validez les modifications :

    Ce code installe et configure à nouveau l’interface CLI Databricks, importe le notebook dans votre système de fichiers Databricks, puis crée et exécute un travail que vous pouvez surveiller dans la page **Workflows** de votre espace de travail. Examinez la sortie et vérifiez que l’exemple de données a été  modifié.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
