---
lab:
  title: Effectuer l’apprentissage d’un modèle d’apprentissage profond
---

# Effectuer l’apprentissage d’un modèle d’apprentissage profond

Dans cet exercice, vous allez utiliser la bibliothèque **PyTorch** pour entraîner un modèle d’apprentissage profond dans Azure Databricks. Ensuite, vous allez utiliser la bibliothèque **Horovod** pour distribuer une formation de Deep Learning sur plusieurs nœuds Worker dans un cluster.

Cet exercice devrait prendre environ **45** minutes.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) dans lequel vous avez un accès administratif.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. Si le script échoue en raison d’un quota ou d’autorisations insuffisants, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

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
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. Pendant que vous attendez, consultez l’article [Entraînement distribué](https://learn.microsoft.com/azure/databricks/machine-learning/train-model/distributed-training/) dans la documentation Azure Databricks.

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
    - **Type de nœud** : Standard_DS3_v2
    - **Arrêter après** *20* **minutes d’inactivité**

1. Attendez que le cluster soit créé. Cette opération peut prendre une à deux minutes.

> **Remarque** : si votre cluster ne démarre pas, le quota de votre abonnement est peut-être insuffisant dans la région où votre espace de travail Azure Databricks est approvisionné. Pour plus d’informations, consultez l’article [La limite de cœurs du processeur empêche la création du cluster](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit). Si cela se produit, vous pouvez essayer de supprimer votre espace de travail et d’en créer un dans une autre région. Vous pouvez spécifier une région comme paramètre pour le script d’installation comme suit : `./mslearn-databricks/setup.ps1 eastus`

## Créer un notebook

Vous allez exécuter du code qui utilise la bibliothèque Spark MLLib pour entraîner un modèle Machine Learning. La première étape consiste donc à créer un notebook dans votre espace de travail.

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
1. Remplacez le nom du notebook par défaut (**Notebook sans titre *[date]***) par **Deep Learning** et, dans la liste déroulante **Connexion**, sélectionnez votre cluster s’il n’est pas déjà sélectionné. Si le cluster n’est pas en cours d’exécution, le démarrage peut prendre une minute.

## Ingérer et préparer les données

Le scénario de cet exercice est basé sur des observations de manchots en Antarctique. L’objectif est de former un modèle Machine Learning pour prédire l’espèce d’un manchot observé en fonction de sa localisation et de ses mensurations corporelles.

> **Citation** : Le jeu de données sur les manchots utilisé dans cet exercice est un sous-ensemble des données collectées et publiées par [ Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) et la [station Palmer en Antarctique](https://pal.lternet.edu/), qui fait partie du [réseau mondial de recherche écologique à long terme (LTER)](https://lternet.edu/).

1. Dans la première cellule du notebook, entrez le code suivant, qui utilise des commandes d’*interpréteur de commandes* pour télécharger les données relatives aux manchots à partir de GitHub dans le système de fichiers utilisé par votre cluster.

    ```bash
    %sh
    rm -r /dbfs/deepml_lab
    mkdir /dbfs/deepml_lab
    wget -O /dbfs/deepml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. Utilisez l’option de menu **&#9656; Exécuter la cellule** à gauche de la cellule pour l’exécuter. Attendez ensuite que le travail Spark s’exécute par le code.
1. Préparez maintenant les données pour le Machine Learning. Sous la cellule de code existante, sélectionnez l’icône **+** pour ajouter une nouvelle cellule de code. Ensuite, dans la nouvelle cellule, entrez et exécutez le code suivant pour :
    - Supprimer toutes les lignes incomplètes
    - Encoder le nom de l’île (chaîne) en tant qu’entier
    - Appliquer les types de données appropriés
    - Normaliser les données numériques à une échelle similaire
    - Fractionnez les données en deux jeux de données : un pour l’entraînement et un autre pour les tests.

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   from sklearn.model_selection import train_test_split
   
   # Load the data, removing any incomplete rows
   df = spark.read.format("csv").option("header", "true").load("/deepml_lab/penguins.csv").dropna()
   
   # Encode the Island with a simple integer index
   # Scale FlipperLength and BodyMass so they're on a similar scale to the bill measurements
   islands = df.select(collect_set("Island").alias('Islands')).first()['Islands']
   island_indexes = [(islands[i], i) for i in range(0, len(islands))]
   df_indexes = spark.createDataFrame(island_indexes).toDF('Island', 'IslandIdx')
   data = df.join(df_indexes, ['Island'], 'left').select(col("IslandIdx"),
                      col("CulmenLength").astype("float"),
                      col("CulmenDepth").astype("float"),
                      (col("FlipperLength").astype("float")/10).alias("FlipperScaled"),
                       (col("BodyMass").astype("float")/100).alias("MassScaled"),
                      col("Species").astype("int")
                       )
   
   # Oversample the dataframe to triple its size
   # (Deep learning techniques like LOTS of data)
   for i in range(1,3):
       data = data.union(data)
   
   # Split the data into training and testing datasets   
   features = ['IslandIdx','CulmenLength','CulmenDepth','FlipperScaled','MassScaled']
   label = 'Species'
      
   # Split data 70%-30% into training set and test set
   x_train, x_test, y_train, y_test = train_test_split(data.toPandas()[features].values,
                                                       data.toPandas()[label].values,
                                                       test_size=0.30,
                                                       random_state=0)
   
   print ('Training Set: %d rows, Test Set: %d rows \n' % (len(x_train), len(x_test)))
    ```

## Installer et importer les bibliothèques PyTorch

PyTorch est une infrastructure permettant de créer des modèles Machine Learning, y compris des réseaux neuronaux profonds (DNN). Étant donné que nous prévoyons d’utiliser PyTorch pour créer notre classifieur de manchot, nous devons importer les bibliothèques PyTorch que nous avons l’intention d’utiliser. PyTorch est déjà installé sur des clusters Azure Databricks avec un runtime ML Databricks (l’installation spécifique de PyTorch dépend du fait que le cluster dispose d’unités de traitement graphique (GPU) qui peuvent être utilisées pour le traitement hautes performances via *cuda*).

1. Ajoutez une nouvelle cellule de code, puis exécutez le code suivant pour préparer l’utilisation de PyTorch :

    ```python
   import torch
   import torch.nn as nn
   import torch.utils.data as td
   import torch.nn.functional as F
   
   # Set random seed for reproducability
   torch.manual_seed(0)
   
   print("Libraries imported - ready to use PyTorch", torch.__version__)
    ```

## Créer des chargeurs de données

PyTorch utilise des *chargeurs de données* pour charger des données d’entraînement et de validation par lots. Nous avons déjà chargé les données dans des tableaux numpy, mais nous devons les envelopper dans les jeux de données PyTorch (dans lesquels les données sont converties en objets pyTorch *tensor*) et créer des chargeurs pour lire des lots à partir de ces jeux de données.

1. Ajoutez une cellule et exécutez le code suivant pour préparer les chargeurs de données :

    ```python
   # Create a dataset and loader for the training data and labels
   train_x = torch.Tensor(x_train).float()
   train_y = torch.Tensor(y_train).long()
   train_ds = td.TensorDataset(train_x,train_y)
   train_loader = td.DataLoader(train_ds, batch_size=20,
       shuffle=False, num_workers=1)

   # Create a dataset and loader for the test data and labels
   test_x = torch.Tensor(x_test).float()
   test_y = torch.Tensor(y_test).long()
   test_ds = td.TensorDataset(test_x,test_y)
   test_loader = td.DataLoader(test_ds, batch_size=20,
                                shuffle=False, num_workers=1)
   print('Ready to load data')
    ```

## Définir un réseau neuronal

Nous sommes maintenant prêts à définir notre réseau neuronal. Dans ce cas, nous allons créer un réseau composé de 3 couches entièrement connectées :

- Une couche d’entrée qui reçoit une valeur d’entrée pour chaque caractéristique (dans ce cas, l’index de l’île et les quatre mesures des manchots) et génère 10 sorties.
- Une couche masquée qui reçoit dix entrées de la couche d’entrée et envoie dix sorties à la couche suivante.
- Une couche de sortie qui génère un vecteur de probabilités pour chacune des trois espèces de manchots possibles.

Au fur et à mesure que nous entraînons le réseau en y faisant passer des données, la fonction **transférer** appliquera les fonctions d’activation *RELU* aux deux premières couches (pour limiter les résultats aux nombres positifs) et retournera une couche de sortie finale qui utilise une fonction *log_softmax* pour renvoyer une valeur qui représente un score de probabilité pour chacune des trois classes possibles.

1. Exécutez le code suivant pour définir le réseau neuronal :

    ```python
   # Number of hidden layer nodes
   hl = 10
   
   # Define the neural network
   class PenguinNet(nn.Module):
       def __init__(self):
           super(PenguinNet, self).__init__()
           self.fc1 = nn.Linear(len(features), hl)
           self.fc2 = nn.Linear(hl, hl)
           self.fc3 = nn.Linear(hl, 3)
   
       def forward(self, x):
           fc1_output = torch.relu(self.fc1(x))
           fc2_output = torch.relu(self.fc2(fc1_output))
           y = F.log_softmax(self.fc3(fc2_output).float(), dim=1)
           return y
   
   # Create a model instance from the network
   model = PenguinNet()
   print(model)
    ```

## Créer des fonctions pour entraîner et tester un modèle de réseau neuronal

Pour entraîner le modèle, nous devons faire passer les valeurs d’entraînement de manière répétée dans le réseau, utiliser une fonction de perte pour calculer la perte, utiliser un optimiseur pour rétropropager les ajustements des valeurs de pondération et de biais, et valider le modèle en utilisant les données de test que nous avons retenues.

1. Pour ce faire, utilisez le code suivant pour créer une fonction pour entraîner et optimiser le modèle et une fonction pour tester le modèle.

    ```python
   def train(model, data_loader, optimizer):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Set the model to training mode
       model.train()
       train_loss = 0
       
       for batch, tensor in enumerate(data_loader):
           data, target = tensor
           #feedforward
           optimizer.zero_grad()
           out = model(data)
           loss = loss_criteria(out, target)
           train_loss += loss.item()
   
           # backpropagate adjustments to the weights
           loss.backward()
           optimizer.step()
   
       #Return average loss
       avg_loss = train_loss / (batch+1)
       print('Training set: Average loss: {:.6f}'.format(avg_loss))
       return avg_loss
              
               
   def test(model, data_loader):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Switch the model to evaluation mode (so we don't backpropagate)
       model.eval()
       test_loss = 0
       correct = 0
   
       with torch.no_grad():
           batch_count = 0
           for batch, tensor in enumerate(data_loader):
               batch_count += 1
               data, target = tensor
               # Get the predictions
               out = model(data)
   
               # calculate the loss
               test_loss += loss_criteria(out, target).item()
   
               # Calculate the accuracy
               _, predicted = torch.max(out.data, 1)
               correct += torch.sum(target==predicted).item()
               
       # Calculate the average loss and total accuracy for this epoch
       avg_loss = test_loss/batch_count
       print('Validation set: Average loss: {:.6f}, Accuracy: {}/{} ({:.0f}%)\n'.format(
           avg_loss, correct, len(data_loader.dataset),
           100. * correct / len(data_loader.dataset)))
       
       # return average loss for the epoch
       return avg_loss
    ```

## Effectuer l’apprentissage d’un modèle

Vous pouvez maintenant utiliser les fonctions **entraîner** et **tester** pour entraîner un modèle de réseau neuronal. Vous entraînez des réseaux neuronaux de manière itérative sur plusieurs *époques*, en enregistrant les statistiques de perte et de précision pour chaque époque.

1. Utilisez le code suivant pour entraîner le modèle :

    ```python
   # Specify the loss criteria (we'll use CrossEntropyLoss for multi-class classification)
   loss_criteria = nn.CrossEntropyLoss()
   
   # Use an optimizer to adjust weights and reduce loss
   learning_rate = 0.001
   optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
   optimizer.zero_grad()
   
   # We'll track metrics for each epoch in these arrays
   epoch_nums = []
   training_loss = []
   validation_loss = []
   
   # Train over 100 epochs
   epochs = 100
   for epoch in range(1, epochs + 1):
   
       # print the epoch number
       print('Epoch: {}'.format(epoch))
       
       # Feed training data into the model
       train_loss = train(model, train_loader, optimizer)
       
       # Feed the test data into the model to check its performance
       test_loss = test(model, test_loader)
       
       # Log the metrics for this epoch
       epoch_nums.append(epoch)
       training_loss.append(train_loss)
       validation_loss.append(test_loss)
    ```

    Pendant que le processus d’entraînement est en cours d’exécution, essayons de comprendre ce qui se passe :

    - Dans chaque *époque*, l’ensemble complet de données d’apprentissage est transmis via le réseau. Il existe cinq caractéristiques pour chaque observation et cinq nœuds correspondants dans la couche d’entrée. Par conséquent, les caractéristiques de chaque observation sont passées sous forme de vecteur de cinq valeurs à cette couche. Toutefois, par souci d’efficacité, les vecteurs de caractéristiques sont regroupés par lots. Ainsi, une matrice de plusieurs vecteurs de caractéristiques est passée à chaque fois.
    - La matrice des valeurs de caractéristiques est traitée par une fonction qui effectue une somme pondérée à l’aide de valeurs de pondération et de biais initialisées. Le résultat de cette fonction est ensuite traité par la fonction d’activation de la couche d’entrée pour limiter les valeurs transmises aux nœuds de la couche suivante.
    - Les fonctions de somme pondérée et d’activation sont répétées dans chaque couche. Notez que les fonctions fonctionnent sur des vecteurs et des matrices plutôt que sur des valeurs scalaires individuelles. En d’autres termes, la transmission de données est essentiellement une série de fonctions d’algèbre linéaire imbriquée. C’est la raison pour laquelle les scientifiques des données préfèrent utiliser des ordinateurs avec des unités de traitement graphique (GPU), car ceux-ci sont optimisés pour les calculs de matrice et de vecteur.
    - Dans la couche finale du réseau, les vecteurs de sortie contiennent une valeur calculée pour chaque classe possible (dans ce cas, les classes 0, 1 et 2). Ce vecteur est traité par une *fonction de perte* qui détermine la distance qui les sépare des valeurs attendues sur la base des classes réelles. Par exemple, supposons que la sortie d’une observation d’un manchot papou (classe 1) soit \[0,3, 0,4, 0,3\]. La prédiction correcte serait \[0,0, 1,0, 0,0\], donc la variance entre les valeurs prédites et réelles (la distance entre chaque valeur prédite et ce qu’elle devrait être) est \[0,3, 0,6, 0,3\]. Cette variance est agrégée pour chaque lot et conservée en tant qu’agrégat en cours d’exécution pour calculer le niveau global d’erreur (*perte*) encouru par les données d’apprentissage pour l’époque.
    - À la fin de chaque époque, les données de validation sont transmises via le réseau, et leur perte et leur précision (proportion de prédictions correctes basées sur la valeur de probabilité la plus élevée dans le vecteur de sortie) sont également calculées. Il est utile de le faire, car il nous permet de comparer les performances du modèle après chaque époque à l’aide de données sur lesquelles il n’a pas été entraîné, nous aidant à déterminer s’il généralisera correctement pour les nouvelles données ou s’il est *surajusté* aux données d’entraînement.
    - Une fois que toutes les données ont été transmises via le réseau, la sortie de la fonction de perte pour les *données* d’entraînement (mais <u>pas</u> les données de *validation*) est transmise à l’optimiseur. Les détails précis de la façon dont l’optimiseur traite la perte varient en fonction de l’algorithme d’optimisation spécifique utilisé. Mais, fondamentalement, vous pouvez considérer l’ensemble du réseau, de la couche d’entrée à la fonction de perte comme étant une grande fonction imbriquée (*composite*). L’optimiseur applique un calcul différentiel pour calculer des *dérivées partielles* pour la fonction en ce qui concerne chaque valeur de pondération et de biais utilisée dans le réseau. Il est possible d’effectuer cette opération efficacement pour une fonction imbriquée en raison d’un élément appelé la *règle de chaîne*, ce qui vous permet de déterminer la dérivée d’une fonction composite à partir des dérivées de sa fonction interne et de ses fonctions externes. Vous n’avez pas vraiment besoin de vous soucier des détails mathématiques ici (l’optimiseur le fait pour vous), mais le résultat final est que les dérivées partielles nous parlent de la pente (ou *dégradé*) de la fonction de perte en ce qui concerne chaque valeur de pondération et de biais. En d’autres termes, nous pouvons déterminer s’il faut augmenter ou diminuer les valeurs de pondération et de biais afin de réduire la perte.
    - Ayant déterminé dans quelle direction ajuster les pondérations et les biais, l’optimiseur utilise le *taux d’apprentissage* pour déterminer à quel point les ajuster, puis il remonte le réseau par un processus appelé *rétropropagation* pour affecter de nouvelles valeurs aux pondérations et aux biais dans chaque couche.
    - Maintenant, l’époque suivante répète l’ensemble du processus d’entraînement, de validation et de rétropropagation à partir des pondérations et des biais révisés de l’époque précédente, ce qui, espérons-le, entraînera un niveau de perte inférieur.
    - Le processus se poursuit ainsi pour 100 époques.

## Passer en revue la perte de formation et de validation

Une fois l’entraînement terminé, nous pouvons examiner les métriques de perte que nous avons enregistrées lors de l’entraînement et de la validation du modèle. Nous recherchons deux choses :

- La perte doit diminuer avec chaque époque, montrant que le modèle apprend les pondérations et les biais appropriés pour prédire les étiquettes correctes.
- La perte d’entraînement et la perte de validation doivent suivre une tendance similaire, montrant que le modèle n’est pas surajusté aux données d’entraînement.

1. Utilisez le code suivant pour tracer le modèle :

    ```python
   %matplotlib inline
   from matplotlib import pyplot as plt
   
   plt.plot(epoch_nums, training_loss)
   plt.plot(epoch_nums, validation_loss)
   plt.xlabel('epoch')
   plt.ylabel('loss')
   plt.legend(['training', 'validation'], loc='upper right')
   plt.show()
    ```

## Afficher les pondérations et les biais appris

Le modèle entraîné se compose des pondérations et des biais finaux qui ont été déterminés par l’optimiseur pendant l’entraînement. En fonction de notre modèle réseau, nous devons nous attendre aux valeurs suivantes pour chaque couche :

- Couche 1 (*fc1*) : Il existe cinq valeurs d’entrée allant à dix nœuds de sortie. Il doit donc y avoir 10 x 5 valeurs de pondérations et 10 valeurs de biais.
- Couche 2 (*fc2*) : Il existe dix valeurs d’entrée allant à dix nœuds de sortie. Il doit donc y avoir 10 x 10 valeurs de pondérations et 10 valeurs de biais.
- Couche 3 (*fc3*) : Il existe dix valeurs d’entrée allant à trois nœuds de sortie. Il doit donc y avoir 3 x 10 valeurs de pondérations et 3 valeurs de biais.

1. Utilisez le code suivant pour afficher les couches de votre modèle entraîné :

    ```python
   for param_tensor in model.state_dict():
       print(param_tensor, "\n", model.state_dict()[param_tensor].numpy())
    ```

## Enregistrer et utiliser le modèle entraîné

Maintenant que nous avons un modèle entraîné, nous pouvons enregistrer ses pondérations entraînées pour une utilisation ultérieure.

1. Utilisez le code suivant pour enregistrer le modèle :

    ```python
   # Save the model weights
   model_file = '/dbfs/penguin_classifier.pt'
   torch.save(model.state_dict(), model_file)
   del model
   print('model saved as', model_file)
    ```

1. Utilisez le code suivant pour charger les pondérations du modèle et prédire l’espèce d'un nouveau manchot observé :

    ```python
   # New penguin features
   x_new = [[1, 50.4,15.3,20,50]]
   print ('New sample: {}'.format(x_new))
   
   # Create a new model class and load weights
   model = PenguinNet()
   model.load_state_dict(torch.load(model_file))
   
   # Set model to evaluation mode
   model.eval()
   
   # Get a prediction for the new data sample
   x = torch.Tensor(x_new).float()
   _, predicted = torch.max(model(x).data, 1)
   
   print('Prediction:',predicted.item())
    ```

## Distribuer un entraînement avec Horovod

L’entraînement du modèle précédent a été effectué sur un nœud unique du cluster. Dans la pratique, il est généralement préférable de mettre à l’échelle l’apprentissage du modèle de Deep Learning sur plusieurs processeurs (ou de préférence des GPU) sur un seul ordinateur, mais dans certains cas, dans certains cas où vous devez transmettre de grands volumes de données d’apprentissage via plusieurs couches d’un modèle de Deep Learning, vous pouvez obtenir une certaine efficacité en distribuant le travail d’entraînement sur plusieurs nœuds de cluster.

Horovod est une bibliothèque open source que vous pouvez utiliser pour distribuer l’entraînement de Deep Learning sur plusieurs nœuds d’un cluster Spark, tels que ceux provisionnés dans un espace de travail Azure Databricks.

### Créer une fonction d’entraînement

Pour utiliser Horovod, vous encapsulez le code pour configurer les paramètres d’entraînement et appeler votre fonction **entraîner** dans une nouvelle fonction, que vous allez exécuter à l’aide de la classe **HorovodRunner** pour distribuer l’exécution sur plusieurs nœuds. Dans votre fonction wrapper d’entraînement, vous pouvez utiliser différentes classes Horovod pour définir un chargeur de données distribué afin que chaque nœud puisse travailler sur un sous-ensemble du jeu de données global, diffuser l’état initial des pondérations de modèle et de l’optimiseur sur tous les nœuds, identifier le nombre de nœuds utilisés et déterminer le code de nœud en cours d’exécution.

1. Exécutez le code suivant pour créer une fonction qui entraîne un modèle à l’aide d’Horovod :

    ```python
   import horovod.torch as hvd
   from sparkdl import HorovodRunner
   
   def train_hvd(model):
       from torch.utils.data.distributed import DistributedSampler
       
       hvd.init()
       
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       if device.type == 'cuda':
           # Pin GPU to local rank
           torch.cuda.set_device(hvd.local_rank())
       
       # Configure the sampler so that each worker gets a distinct sample of the input dataset
       train_sampler = DistributedSampler(train_ds, num_replicas=hvd.size(), rank=hvd.rank())
       # Use train_sampler to load a different sample of data on each worker
       train_loader = torch.utils.data.DataLoader(train_ds, batch_size=20, sampler=train_sampler)
       
       # The effective batch size in synchronous distributed training is scaled by the number of workers
       # Increase learning_rate to compensate for the increased batch size
       learning_rate = 0.001 * hvd.size()
       optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
       
       # Wrap the local optimizer with hvd.DistributedOptimizer so that Horovod handles the distributed optimization
       optimizer = hvd.DistributedOptimizer(optimizer, named_parameters=model.named_parameters())
   
       # Broadcast initial parameters so all workers start with the same parameters
       hvd.broadcast_parameters(model.state_dict(), root_rank=0)
       hvd.broadcast_optimizer_state(optimizer, root_rank=0)
   
       optimizer.zero_grad()
   
       # Train over 50 epochs
       epochs = 100
       for epoch in range(1, epochs + 1):
           print('Epoch: {}'.format(epoch))
           # Feed training data into the model to optimize the weights
           train_loss = train(model, train_loader, optimizer)
   
       # Save the model weights
       if hvd.rank() == 0:
           model_file = '/dbfs/penguin_classifier_hvd.pt'
           torch.save(model.state_dict(), model_file)
           print('model saved as', model_file)
    ```

1. Utilisez le code suivant pour appeler votre fonction à partir d’un objet **HorovodRunner** :

    ```python
   # Reset random seed for PyTorch
   torch.manual_seed(0)
   
   # Create a new model
   new_model = PenguinNet()
   
   # We'll use CrossEntropyLoss to optimize a multiclass classifier
   loss_criteria = nn.CrossEntropyLoss()
   
   # Run the distributed training function on 2 nodes
   hr = HorovodRunner(np=2, driver_log_verbosity='all') 
   hr.run(train_hvd, model=new_model)
   
   # Load the trained weights and test the model
   test_model = PenguinNet()
   test_model.load_state_dict(torch.load('/dbfs/penguin_classifier_hvd.pt'))
   test_loss = test(test_model, test_loader)
    ```

Il se peut que vous deviez faire défiler l’écran pour voir toute la sortie, qui devrait afficher quelques messages d’information d’Horovod suivis de la sortie enregistrée des nœuds (parce que le paramètre **driver_log_verbosity** est défini sur **tout**). Les sorties de nœud doivent afficher la perte après chaque époque. Enfin, la fonction **tester** est utilisée pour tester le modèle entraîné.

> **Conseil** : Si la perte ne diminue pas après chaque époque, réessayez d’exécuter la cellule !

## Nettoyage

Dans le portail Azure Databricks, sur la page **Calcul**, sélectionnez votre cluster et sélectionnez **&#9632; Arrêter** pour l’arrêter.

Si vous avez terminé d’explorer Azure Databricks, vous pouvez supprimer les ressources que vous avez créées pour éviter les coûts Azure inutiles et libérer de la capacité dans votre abonnement.
