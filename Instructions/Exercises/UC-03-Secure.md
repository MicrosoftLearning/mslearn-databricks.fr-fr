---
lab:
  title: Sécurisation des données dans Unity Catalog
---

# Sécurisation des données dans Unity Catalog

La sécurité des données est une préoccupation essentielle pour les organisations qui gèrent des informations sensibles dans leur data lakehouse. À mesure que les équipes chargées des données collaborent entre différents rôles et services, il devient de plus en plus complexe de garantir que les bonnes personnes aient accès aux bonnes données, tout en protégeant les informations sensibles contre les accès non autorisés.

Ce labo pratique présente deux fonctionnalités de sécurité puissantes de Unity Catalog qui vont au-delà du contrôle d’accès de base :

1. **Filtrage des lignes et masquage des colonnes** : Découvrez comment protéger les données sensibles au niveau des tables en masquant des lignes spécifiques ou des valeurs de colonnes en fonction des autorisations des utilisateurs
2. **Vues dynamiques** : Créez des vues intelligentes qui ajustent automatiquement les données visibles par les utilisateurs selon leurs appartenances à des groupes et leurs niveaux d’accès

Ce labo prend environ **45** minutes.

> **Remarque** : l’interface utilisateur d’Azure Databricks est soumise à une amélioration continue. Elle a donc peut-être changé depuis l’écriture des instructions de cet exercice.

## Provisionner un espace de travail Azure Databricks

> **Conseil** : Si vous disposez déjà d’un espace de travail Azure Databricks, vous pouvez ignorer cette procédure et utiliser votre espace de travail existant.

Cet exercice inclut un script permettant d’approvisionner un nouvel espace de travail Azure Databricks. Le script tente de créer une ressource d’espace de travail Azure Databricks de niveau *Premium* dans une région dans laquelle votre abonnement Azure dispose d’un quota suffisant pour les cœurs de calcul requis dans cet exercice ; et suppose que votre compte d’utilisateur dispose des autorisations suffisantes dans l’abonnement pour créer une ressource d’espace de travail Azure Databricks. 

Si le script échoue en raison d’un quota insuffisant ou d’autorisations insuffisantes, vous pouvez essayer de [créer un espace de travail Azure Databricks de manière interactive dans le portail Azure](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace).

1. Dans un navigateur web, connectez-vous au [portail Azure](https://portal.azure.com) à l’adresse `https://portal.azure.com`.
2. Cliquez sur le bouton **[\>_]** à droite de la barre de recherche, en haut de la page, pour créer un environnement Cloud Shell dans le portail Azure, puis sélectionnez un environnement ***PowerShell***. Cloud Shell fournit une interface de ligne de commande dans un volet situé en bas du portail Azure, comme illustré ici :

    ![Portail Azure avec un volet Cloud Shell](./images/cloud-shell.png)

    > **Remarque** : si vous avez déjà créé un Cloud Shell qui utilise un environnement *Bash*, basculez-le vers ***PowerShell***.

3. Notez que vous pouvez redimensionner Cloud Shell en faisant glisser la barre de séparation en haut du volet. Vous pouvez aussi utiliser les icônes **&#8212;**, **&#10530;** et **X** situées en haut à droite du volet pour réduire, agrandir et fermer ce dernier. Pour plus d’informations sur l’utilisation d’Azure Cloud Shell, consultez la [documentation Azure Cloud Shell](https://docs.microsoft.com/azure/cloud-shell/overview).

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
7. Attendez que le script se termine. Cela prend généralement environ 5 minutes, mais dans certains cas, cela peut prendre plus de temps. En attendant, passez en revue le module d’apprentissage [Mettre en œuvre la sécurité et le contrôle d’accès dans Unity Catalog](https://learn.microsoft.com/training/modules/implement-security-unity-catalog) dans Microsoft Learn.

## Créer un catalogue

1. Connectez-vous à un espace de travail lié au metastore.
2. Sélectionnez **Catalogue** dans le menu de gauche.
3. Sélectionnez **Catalogues** sous **Accès rapide**.
4. Sélectionnez **Créer un catalogue**.
5. Dans la boîte de dialogue **Créer un nouveau catalogue**, entrez un **Nom de catalogue** et sélectionnez le **Type** de catalogue que vous souhaitez créer : Catalogue **standard**.
6. Spécifiez un **Emplacement de stockage** géré.

## Créer un notebook

1. Dans la barre latérale, cliquez sur le lien **(+) Nouveau** pour créer un **notebook**.
   
1. Remplacez le nom par défaut du notebook (**Notebook sans titre *[date]***) par `Secure data in Unity Catalog` et, dans la liste déroulante **Se connecter**, sélectionnez **Cluster serverless** s’il n’est pas déjà sélectionné.  Notez que **Serverless** est activé par défaut.

1. Copiez et exécutez le code suivant dans une nouvelle cellule de votre notebook afin de configurer votre environnement de travail pour ce cours. Cela permettra également de définir votre catalogue par défaut sur votre catalogue spécifique et le schéma sur le nom de schéma indiqué ci-dessous l’aide des instructions `USE`.

```SQL
USE CATALOG `<your catalog>`;
USE SCHEMA `default`;
```

1. Exécutez le code ci-dessous et vérifiez que votre catalogue actuel est défini sur votre nom de catalogue unique et que le schéma actuel est **par défaut**.

```sql
SELECT current_catalog(), current_schema()
```

## Protéger les colonnes et les lignes à l’aide du masquage des colonnes et du filtrage des lignes

### Créer la table Customers

1. Exécutez le code ci-dessous pour créer la table **customers** dans votre schéma **par défaut**.

```sql
CREATE OR REPLACE TABLE customers AS
SELECT *
FROM samples.tpch.customer;
```

2. Exécutez une requête pour afficher *10* lignes de la table **customers** dans votre schéma **par défaut**. Notez que la table contient des informations telles que **c_name**, **c_phone** et **c_mktsegment**.
   
```sql
SELECT *
FROM customers  
LIMIT 10;
```

### Créer une fonction pour effectuer le masquage des colonnes

Consultez la documentation [Filtrer les données sensibles de la table à l’aide de filtres de lignes et de masques de colonnes](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) pour obtenir de l’aide supplémentaire.

1. Créez une fonction nommée **phone_mask** qui effectue le masquage de la colonne **c_phone** dans la table **customers** si l’utilisateur n’est pas membre du groupe (« admin ») à l’aide de la fonction `is_account_group_member`. La fonction **phone_mask** doit renvoyer la chaîne *REDACTED PHONE NUMBER* si l’utilisateur n’est pas membre.

```sql
CREATE OR REPLACE FUNCTION phone_mask(c_phone STRING)
  RETURN CASE WHEN is_account_group_member('metastore_admins') 
    THEN c_phone 
    ELSE 'REDACTED PHONE NUMBER' 
  END;
```

2. Appliquez la fonction de masquage de colonne **phone_mask** à la table **customers** à l’aide de l’instruction `ALTER TABLE`.

```sql
ALTER TABLE customers 
  ALTER COLUMN c_phone 
  SET MASK phone_mask;
```

3. Exécutez la requête ci-dessous pour afficher la table **customers** avec le masquage de colonne appliqué. Vérifiez que la colonne **c_phone** affiche la valeur *REDACTED PHONE NUMBER*.

```sql
SELECT *
FROM customers
LIMIT 10;
```

### Créer une fonction pour effectuer le filtrage des lignes

Consultez la documentation [Filtrer les données sensibles de la table à l’aide de filtres de lignes et de masques de colonnes](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/) pour obtenir de l’aide supplémentaire.

1. Exécutez le code ci-dessous pour compter le nombre total de lignes dans la table **customers**. Vérifiez que la table contient 750 000 lignes de données.

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

2. Créez une fonction nommée **nation_filter** qui filtre sur **c_nationkey** dans la table **customers** si l’utilisateur n’est pas membre du groupe (« admin ») à l’aide de la fonction `is_account_group_member`. La fonction ne doit renvoyer que les lignes où **c_nationkey** est égal à *21*.

    Consultez la documentation [if function](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/if) pour obtenir de l’aide supplémentaire.

```sql
CREATE OR REPLACE FUNCTION nation_filter(c_nationkey INT)
  RETURN IF(is_account_group_member('admin'), true, c_nationkey = 21);
```

3. Appliquez la fonction de filtrage des lignes `nation_filter` à la table **customers** à l’aide de l’instruction `ALTER TABLE`.

```sql
ALTER TABLE customers 
SET ROW FILTER nation_filter ON (c_nationkey);
```

4. Exécutez la requête ci-dessous pour compter le nombre de lignes dans la table **customers** pour vous, car vous avez filtré les lignes pour les utilisateurs qui ne sont pas administrateurs. Vérifiez que vous ne pouvez voir que *29 859* lignes (*où c_nationkey = 21*). 

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

5. Exécutez la requête ci-dessous pour afficher la table **customers**. 

    Confirmez que la table finale :
    - masque la colonne **c_phone** et
    - filtre les lignes selon la colonne **c_nationkey** pour les utilisateurs qui ne sont pas *administrateurs*.

```sql
SELECT *
FROM customers;
```

## Protection des colonnes et des lignes avec des vues dynamiques

### Créer la table Customers_new

1. Exécutez le code ci-dessous pour créer la table **customers_new** dans votre schéma **par défaut**.

```sql
CREATE OR REPLACE TABLE customers_new AS
SELECT *
FROM samples.tpch.customer;
```

2. Exécutez une requête pour afficher *10* lignes de la table **customers_new** dans votre schéma **par défaut**. Notez que la table contient des informations telles que **c_name**, **c_phone** et **c_mktsegment**.

```sql
SELECT *
FROM customers_new
LIMIT 10;
```

### Créer la Vue dynamique

Créons une vue nommée **vw_customers** qui présente une vue traitée des données de la table **customers_new** avec les transformations suivantes :

- Sélectionne toutes les colonnes de la table **customers_new**.

- Masquez toutes les valeurs de la colonne **c_phone** en *REDACTED PHONE NUMBER*, sauf si vous êtes dans le `is_account_group_member('admins')`
    - CONSEIL : Utilisez une instruction `CASE WHEN` dans la clause `SELECT`.

- Limitez les lignes où **c_nationkey** est égal à *21*, sauf si vous êtes dans le `is_account_group_member('admins')`.
    - CONSEIL : Utilisez une instruction `CASE WHEN` dans la clause `WHERE`.

```sql
-- Create a movies_gold view by redacting the "votes" column and restricting the movies with a rating below 6.0

CREATE OR REPLACE VIEW vw_customers AS
SELECT 
  c_custkey, 
  c_name, 
  c_address, 
  c_nationkey,
  CASE 
    WHEN is_account_group_member('admins') THEN c_phone
    ELSE 'REDACTED PHONE NUMBER'
  END as c_phone,
  c_acctbal, 
  c_mktsegment, 
  c_comment
FROM customers_new
WHERE
  CASE WHEN
    is_account_group_member('admins') THEN TRUE
    ELSE c_nationkey = 21
  END;
```

3. Affichez les données dans la vue **vw_customers**. Confirmez que la colonne **c_phone** est masquée. Confirmez que **c_nationkey** est égal à *21*, sauf si vous êtes l’administrateur.

```sql
SELECT * 
FROM vw_customers;
```

6. Comptez le nombre de lignes dans la vue **vw_customers**. Confirmez que la vue contient *29 859* lignes.

```sql
SELECT count(*)
FROM vw_customers;
```

### Accorder les droits d’accès à la vue

1. Accordons aux « utilisateurs de compte » l’accès à la vue **vw_customers**.

**REMARQUE :** Vous devrez également fournir aux utilisateurs un accès au catalogue et au schéma. Dans cet environnement d’entraînement partagé, vous ne pouvez pas accorder l’accès à votre catalogue à d’autres utilisateurs.

```sql
GRANT SELECT ON VIEW vw_customers TO `account users`
```

2. Utilisez l’instruction `SHOW` pour afficher tous les privilèges (hérités, refusés et accordés) qui affectent la vue **vw_customers**. Vérifiez que la colonne **Principal** contient des *utilisateurs de compte*.

    Consultez la documentation [SHOW GRANTS](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/security-show-grant) pour obtenir de l’aide.

```sql
SHOW GRANTS ON vw_customers
```