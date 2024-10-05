---
lab:
  title: Entraîner un modèle avec AutoML
---

# Entraîner un modèle avec AutoML

AutoML est une fonctionnalité d’Azure Databricks qui tente plusieurs algorithmes et paramètres avec vos données pour entraîner un modèle Machine Learning optimal.

Cet exercice devrait prendre environ **30** minutes.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner un espace de travail Azure Databricks

> **Remarque** : Pour cet exercice, vous avez besoin d’un espace de travail Azure Databricks **Premium** dans une région où la *mise à disposition de modèles* est prise en charge. Pour plus d’informations sur les fonctionnalités régionales d’Azure Databricks, consultez [Régions Azure Databricks](https://learn.microsoft.com/azure/databricks/resources/supported-regions). Si vous disposez déjà d’un espace de travail Azure Databricks de type *Premium* ou *Essai* dans une région appropriée, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice comprend un script qui permet de provisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Qu’est-ce qu’AutoML ?](https://learn.microsoft.com/azure/databricks/machine-learning/automl/) dans la documentation Azure Databricks.

## Créer un cluster

Azure Databricks est une plateforme de traitement distribuée qui utilise des *clusters Apache Spark* pour traiter des données en parallèle sur plusieurs nœuds. Chaque cluster se compose d’un nœud de pilote pour coordonner le travail et les nœuds Worker pour effectuer des tâches de traitement. Dans cet exercice, vous allez créer un cluster à *nœud unique* pour réduire les ressources de calcul utilisées dans l’environnement du labo (dans lequel les ressources peuvent être limitées). Dans un environnement de production, vous créez généralement un cluster avec plusieurs nœuds Worker.

> **Conseil** : Si vous disposez déjà d’un cluster avec une version 13.3 LTS **<u>ML</u>** ou ultérieure du runtime dans votre espace de travail Azure Databricks, vous pouvez l’utiliser pour effectuer cet exercice et ignorer cette procédure.

1. Dans le portail Microsoft Azure, accédez au groupe de ressources **msl-*xxxxxxx*** créé par le script (ou le groupe de ressources contenant votre espace de travail Azure Databricks existant)
1. Sélectionnez votre ressource de service Azure Databricks (nommée **databricks-*xxxxxxx*** si vous avez utilisé le script d’installation pour la créer).
1. Dans la page **Vue d’ensemble** de votre espace de travail, utilisez le bouton **Lancer l’espace de travail** pour ouvrir votre espace de travail Azure Databricks dans un nouvel onglet de navigateur et connectez-vous si vous y êtes invité.

    > **Conseil** : lorsque vous utilisez le portail de l’espace de travail Databricks, plusieurs conseils et notifications peuvent s’afficher. Ignorez-les et suivez les instructions fournies pour effectuer les tâches de cet exercice.

1. Dans la barre latérale située à gauche, sélectionnez la tâche **(+) Nouveau**, puis sélectionnez **Cluster**.
1. Dans la page **Nouveau cluster**, créez un cluster avec les paramètres suivants :
    - **Nom du cluster** : cluster de *nom d’utilisateur* (nom de cluster par défaut)
    - **Stratégie** : Non restreint
    - **Mode cluster** : nœud unique
    - **Mode d’accès** : un seul utilisateur (*avec votre compte d’utilisateur sélectionné*)
    - **Version du runtime Databricks** : *Sélectionnez l’édition **<u>ML</u>** de la dernière version non bêta du runtime (**Not** version du runtime standard) qui :*
        - *N’utilise **pas** de GPU*
        - *Inclut Scala > **2.11***
        - *Inclut Spark > **3.4***
    - **Utiliser l’accélération photon** : <u>Non</u> sélectionné
    - **Type de nœud** : Standard_D4ds_v5
    - **Arrêter après** *20* **minutes d’inactivité**

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Charger des données d’apprentissage dans un entrepôt SQL

Pour entraîner un modèle Machine Learning à l’aide d’AutoML, vous devez charger les données d’apprentissage. Dans cet exercice, vous allez entraîner un modèle à classer un manchot dans l’une des trois espèces sur la base d’observations comprenant sa localisation et ses mesures corporelles. Vous allez charger des données d’apprentissage qui incluent l’étiquette d’espèce dans une table dans un entrepôt de données Azure Databricks.

1. Dans le portail Azure Databricks de votre espace de travail, dans la barre latérale, sous **SQL**, sélectionnez **Entrepôts SQL**.
1. Notez que l’espace de travail inclut déjà un entrepôt SQL nommé **Starter Warehouse**.
1. Dans le menu **Actions** (**⁝**) de l’entrepôt SQL, sélectionnez **Modifier**. Définissez ensuite la propriété **Taille du cluster** sur **2X-Small** et enregistrez vos modifications.
1. Sélectionnez le bouton **Démarrer** pour lancer l’entrepôt SQL (ce qui peut prendre une minute ou deux).

> **Remarque** : si votre entrepôt SQL ne démarre pas, il se peut que le quota de votre abonnement soit insuffisant dans la région où votre espace de travail Azure Databricks est configuré. Consultez l’article [Quota de processeurs virtuels Azure requis](https://docs.microsoft.com/azure/databricks/sql/admin/sql-endpoints#required-azure-vcpu-quota) pour plus de détails. Si cela se produit, vous pouvez essayer de demander une augmentation de quota comme indiqué dans le message d’erreur lorsque l’entrepôt ne parvient pas à démarrer. Vous pouvez également essayer de supprimer votre espace de travail et d’en créer un autre dans une région différente. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

1. Téléchargez le fichier [**penguins.csv**](https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv) à partir de `https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv` vers votre ordinateur local, en l’enregistrant en tant que **penguins.csv**.
1. Dans le portail de l’espace de travail Azure Databricks, dans la barre latérale, sélectionnez **(+) Nouveau**, puis **Chargement de fichiers** et chargez le fichier **pingouins.csv** que vous avez téléchargé sur votre ordinateur.
1. Dans la page **Charger des données**, sélectionnez le schéma **par défaut** et définissez le nom de la table sur **manchots**. Sélectionnez ensuite **Créer une table** dans le coin inférieur gauche de la page.
1. Une fois la table créée, vérifiez ses détails.

## Créer une expérience AutoML

Maintenant que vous avez des données, vous pouvez les utiliser avec AutoML pour entraîner un modèle.

1. Dans la barre latérale de gauche, sélectionnez **Expériences**.
1. Dans la page **Expériences**, sélectionnez **Créer une expérience AutoML**.
1. Configurez l’expérience AutoML avec les paramètres suivants :
    - **Cluster** : *Sélectionner votre cluster*
    - **Type de problème ML :** Classification
    - **Jeu de données d’apprentissage d’entrée** : *Accéder à la base de données **par défaut** et sélectionner la table **manchots***
    - **Cible de prédiction** : Espèce
    - **Nom de l’expérience** : Penguin-classification
    - **Configuration avancée** :
        - **Métrique d’évaluation** : Précision
        - **Infrastructures d’entraînement** : lightgbm, sklearn, xgboost
        - **Timeout** (Expiration du délai) : 5
        - **Colonne d’heure pour l’entraînement/validation/test de fractionnement** : *Laisser vide*
        - **Étiquette positive** : *Laisser vide*
        - **Emplacement de stockage des données intermédiaire** : Artefact MLflow
1. Utilisez le bouton **Démarrer AutoML** pour démarrer l’expérience. Fermez les boîtes de dialogue d’informations affichées.
1. Attendez la fin de l’expérience. Vous pouvez utiliser le bouton **Actualiser** à droite pour afficher les détails des exécutions générées.
1. Après cinq minutes, l’expérience se termine. L’actualisation des exécutions affiche l’exécution qui a entraîné le modèle le plus performant (en fonction de la métrique de *précision* que vous avez sélectionnée) en haut de la liste.

## Déployer le modèle le plus performant

Après avoir exécuté une expérience AutoML, vous pouvez explorer le modèle le plus performant qu’il a généré.

1. Dans la page d’expérience **Penguin-classification**, sélectionnez **Afficher le notebook pour le meilleur modèle** pour ouvrir le notebook utilisé pour entraîner le modèle dans un nouvel onglet de navigateur.
1. Faites défiler les cellules du notebook, en notant le code utilisé pour entraîner le modèle.
1. Fermez l’onglet du navigateur contenant le notebook pour revenir à la page d’expérience **Penguin-classification**.
1. Dans la liste des exécutions, sélectionnez le nom de la première exécution (qui a produit le meilleur modèle) pour l’ouvrir.
1. Dans la section **Artefacts**, notez que le modèle a été enregistré en tant qu’artefact MLflow. Utilisez ensuite le bouton **Inscrire le modèle** pour inscrire le modèle en tant que nouveau modèle nommé **Penguin-Classifier**.
1. Dans la barre latérale située à gauche, passez à la page **Modèles**. Sélectionnez ensuite le modèle **Penguin-Classifier** que vous venez d’inscrire.
1. Sur la page **Penguin-Classifier**, utilisez le bouton **Utiliser le modèle pour l’inférence** pour créer un point de terminaison en temps réel avec les paramètres suivants :
    - **Modèle** : Penguin-Classifier
    - **Version du modèle** : 1
    - **Point de terminaison**: classify-penguin
    - **Taille de calcul** : Small

    Le point de terminaison de mise à disposition est hébergé dans un nouveau cluster, dont la création peut prendre plusieurs minutes.
  
1. Une fois le point de terminaison créé, utilisez le bouton **Interroger le point de terminaison** en haut à droite pour ouvrir une interface à partir de laquelle vous pouvez tester le point de terminaison. Dans l’interface de test, sous l’onglet **Navigateur**, entrez la requête JSON suivante, puis utilisez le bouton **Envoyer la requête** pour appeler le point de terminaison et générer une prédiction.

    ```json
    {
      "dataframe_records": [
      {
         "Island": "Biscoe",
         "CulmenLength": 48.7,
         "CulmenDepth": 14.1,
         "FlipperLength": 210,
         "BodyMass": 4450
      }
      ]
    }
    ```

1. Faites des essais en utilisant différentes valeurs relatives aux caractéristiques des manchots, puis observez les résultats retournés. Fermez ensuite l’interface de test.

## Supprimer le point de terminaison

Lorsque le point de terminaison n’est plus nécessaire, vous devez le supprimer pour éviter les coûts inutiles.

Dans la page du point de terminaison **classify-penguin**, dans le menu **&#8285;**, sélectionnez **Supprimer**.

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé l’exploration d’Azure Databricks, vous pouvez supprimer les ressources que vous avez créées afin d’éviter des coûts Azure non nécessaires et de libérer de la capacité dans votre abonnement.

> **Informations complémentaires** : Pour plus d’informations, consultez [Comment fonctionne Databricks AutoML](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/automl/how-automl-works) dans la documentation Azure Databricks.
