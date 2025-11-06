# 🏠 Airbnb Analytics — Amsterdam Insights with dbt
## 📖 Contexte & objectifs

Ce projet a pour objectif d’analyser le marché Airbnb à Amsterdam à partir de données ouvertes, en construisant un pipeline analytique reproductible avec dbt (Data Build Tool) et Snowflake.

L’ambition est de transformer des données brutes en indicateurs exploitables pour mieux comprendre les dynamiques du marché de la location courte durée et les tendances touristiques associées.

## 🎯 Questions analytiques
### 🧩 Objectif 1 — Comprendre le marché Airbnb à Amsterdam

🏙 Quelle est la distribution des prix par quartier ?

⭐ Comment se répartissent les super-hôtes dans la ville ?

💰 Existe-t-il une corrélation entre le statut de super-hôte et le prix moyen des annonces ?

### 🌍 Objectif 2 — Étudier les tendances touristiques

En combinant les jeux de données :

curation_tourists_per_year (nombre de touristes par an),

et les données de reviews (commentaires Airbnb par année),

le projet vise à :

📊 déterminer si les touristes privilégient davantage Airbnb que les hôtels,

📈 observer l’évolution de cette tendance au fil des années.

## ⚙️ Stack technique
| Outil                     | Rôle                                              |
| ------------------------- | ------------------------------------------------- |
| **dbt**                   | Transformation et modélisation des données        |
| **Snowflake**             | Entrepôt de données                               |
| **GitHub**                | Versioning et documentation du projet             |
| **Inside Airbnb Dataset** | Source des données (hôtes, logements, prix, etc.) |

# Le jeu de données Airbnb
## Source: 
Le jeu de données a été téléchargé depuis le site https://insideairbnb.com/get-the-data/ qui regroupe les données Airbnb 
pour plusieurs villes. Pour notre travail, nous avons choisi la ville d'Amsterdam correspondant à un extrait du 11 Mars 2024

<details>
   <summary>Mise en place de l’environnement</summary>

## C'est quoi DBT?
   DBT est un outil SQL qui permet:
1. Aux DA/DE d'écrire les trasnformations de leurs données en SQL
2. Aux DA/DE de ne âs répéter leur code SQL grâce à la modularisation et à la paramétrisation
3. Aux DA/DE de tester leurs code SQL en isolation mais aussi de voir comment il s'intègre à la plateforme analytics
4. Aux entreprises de s'assurer de la traçabilité des données
5. Aux entreprises d'appliquer les bonnes règles de data gouvernance

## Traitement
1. Division du fichier `listings.csv.gz` en 2 fichiers:
   1. [listings](dataset/listings.csv) avec un nombre de colonnes réduit et qui ne contient que les donneées qui 
   touchent directement au listing (i.e. on a enlevé les données de l'hôte et sur les revues) 
   2. [hosts](dataset/hosts.csv) ce fichier, extrait du fichier `listings.csv.gz`, ne contient que les infos concernant 
   l'hoôte. Ici aussi, nous avons limité le nombre de colonnes par rapport à toutes les infos qu'on avait
2. [reviews](dataset/reviews.csv) ce fichier a été téléchargé du jeu de données résumé où on n'a que 2 colonnes:
le `listing_id` et la `date` du commentaire qui a été laissé. Par exemple, ces 2 lignes ci-dessous
```csv
262394,2012-04-11
262394,2012-04-25
```
indiquent que le `listing_id` 262394 a reçu 2 commentaires: un le 11 Avril 2012 et l'autre le 25 Avril 2012.

## Configuration de Snowflake
Exécutez le script suivant dans Snowflake pour configurer le rôle, l’utilisateur dbt et les accès nécessaires :
```sql
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS transform;

GRANT ROLE TRANSFORM TO ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;

GRANT OPERATE ON WAREHOUSE COMPUTE_WH TO ROLE transform;

CREATE USER IF NOT EXISTS dbt
    PASSWORD='MotDePasseDBT123@'
    LOGIN_NAME='dbt'
    MUST_CHANGE_PASSWORD=FALSE
    DEFAULT_WAREHOUSE='COMPUTE_WH'
    DEFAULT_ROLE='transform'
    DEFAULT_NAMESPACE='AIRBNB_PROJECT.RAW'
    COMMENT='Utilisateur DBT pour la transformation des données';
GRANT ROLE transform TO user dbt;

CREATE DATABASE IF NOT EXISTS AIRBNB_PROJECT;
CREATE SCHEMA IF NOT EXISTS AIRBNB_PROJECT.RAW;

GRANT ALL ON WAREHOUSE COMPUTE_WH TO ROLE transform; 
GRANT ALL ON DATABASE AIRBNB_PROJECT to ROLE transform;
GRANT ALL ON ALL SCHEMAS IN DATABASE AIRBNB_PROJECT to ROLE transform;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE AIRBNB_PROJECT to ROLE transform;
GRANT ALL ON ALL TABLES IN SCHEMA AIRBNB_PROJECT.RAW to ROLE transform;
GRANT ALL ON FUTURE TABLES IN SCHEMA AIRBNB_PROJECT.RAW to ROLE transform;
```

💡 Tous les scripts SQL détaillés sont disponibles dans le dépôt (CREATION ET MISE EN PLACE DE DBT SUR SNOWFLAKE.sql).
## Chargement des donneées
Les données sont chargées directement depuis le dépôt GitHub à l’aide de l’intégration git_https_api.
Le schéma RAW contient trois tables sources :

HOSTS

LISTINGS

REVIEWS

Les scripts complets de création et d’insertion se trouvent dans le fichier
CREATION TABLE SUR SNOWFLAKE.sql.
</details>

<details>
   <summary>Création des modèles dbt</summary>
  Les modèles dbt définissent la logique de transformation SQL appliquée aux données brutes.
Chaque modèle génère une vue ou table nettoyée, stockée dans le schéma curation.
✳️ Exemple : curation_hosts.sql
	
```snowflake
WITH hosts_raw AS (
    SELECT
		host_id,
		CASE WHEN len(host_name) = 1 THEN 'Anonyme' ELSE host_name END AS host_name,
		host_since,
		host_location,
		SPLIT_PART(host_location, ',', 1) AS host_city,
		SPLIT_PART(host_location, ',', 2) AS host_country,
		TRY_CAST(REPLACE(host_response_rate, '%', '') AS INTEGER) AS response_rate,
		host_is_superhost = 't' AS is_superhost,
		host_neighbourhood,
		host_identity_verified = 't' AS is_identity_verified
    FROM airbnb.raw.hosts)
SELECT *
from hosts_raw
```

💰 Exemple : curation_listings.sql
```snowflake
WITH listings_raw AS 
	(SELECT 
		id AS listing_id,
		listing_url,
		name,
		description,
		description IS NOT NULL has_description,
		neighbourhood_overview,
		neighbourhood_overview IS NOT NULL AS has_neighrbourhood_description,
		host_id,
		latitude,
		longitude,
		property_type,
		room_type,
		accommodates,
		bathrooms,
		bedrooms,
		beds,
		amenities,
        try_cast(split_part(price, '$', 1) as float) as price,
		minimum_nights,
		maximum_nights
	FROM airbnb.raw.listings )
SELECT *
FROM listings_raw
```

Le SQL pour `curation_reviews` est le suivant:
```
WITH curation_reviews as 
(
    SELECT 
        LISTING_ID,
        DATE review_date
    FROM
        airbnb.raw.reviews
)

SELECT
     LISTING_ID,
    review_date,
    COUNT(*) number_reviews
FROM 
    curation_reviews
GROUP BY LISTING_ID, review_date

```
<img width="677" height="502" alt="image" src="https://github.com/user-attachments/assets/9fb4c2d0-664b-4ca8-89e7-45a237966ae9" />
Nos models CURATION sont la version "propre" de nos donnée provenant de la source principale RAW

🧮 Macros dbt
🎯 Macro : extraire_prix_a_partir_dun_caractere.sql
Cette macro permet de convertir une chaîne de caractères en valeur numérique,
en gérant les cas où le symbole $ se trouve avant ou après le montant.

```
{% macro extraire_prix_a_partir_dun_caractere(price, symbol) -%}
    try_cast(
        CASE 
            WHEN STARTSWITH({{ price }}, '{{ symbol }}') THEN SPLIT_PART({{ price }}, '{{ symbol }}', 2)
            WHEN ENDSWITH({{ price }}, '{{ symbol }}') THEN SPLIT_PART({{ price }}, '{{ symbol }}', 1)
            ELSE NULL
        END
    AS FLOAT)
{% endmacro %}

```
Utilisation: 
```
{{ extraire_prix_a_partir_dun_caractere('price', '$') }} AS price

```
</details>
<details>
	<summary>Définition des sources</summary>
	Le fichier sources.yaml définit les dépendances entre les tables sources et les modèles dbt :
	
	```
	version: 2

	sources:
	  - name: raw_airbnb_data
	    database: airbnb_project
	    schema: raw
	    tables:
	      - name: hosts
	      - name: listings
	      - name: reviews

	```
</details>
<details>
	<summary>Seeds</summary>
	Un seed est utilisé pour intégrer des données statiques dans dbt, ici tourists_per_year.csv
	
	```
	seeds:
  	airbnb_analytics:
    tourists_per_year:
      +enabled: true
      +database: airbnb_project
      +schema: raw
	```
Transformation en model curation:
```
WITH curation_tourists AS (
    SELECT  
        year,
        tourists
    FROM {{ ref('tourists_per_year') }}
)
SELECT  
    DATE(year || '-12-31') AS year,
    tourists
FROM curation_tourists;

```
</details>	
<details>
	<summary>Snapshots</summary>
	Les snapshots permettent de suivre l’évolution des données dans le temps.
	hosts_snapshot.sql
	
	```
		{% snapshot hosts_snapshot %}
	    {{
	        config(
	          target_database='airbnb',
	          target_schema='snapshots',
	          strategy='check',
	          check_cols='all',
	          unique_key='host_id'
	        )
	    }}
	    SELECT * FROM {{ source('raw_airbnb_data', 'hosts') }}
		{% endsnapshot %}
	```
	
listings_snapshot.sql
```
{% snapshot listings_snapshot %}
    {{
        config(
          target_database='airbnb',
          target_schema='snapshots',
          strategy='check',
          check_cols='all',
          unique_key='id'
        )
    }}
    SELECT * FROM {{ source('raw_airbnb_data', 'listings') }}
{% endsnapshot %}

```
</details>	
<details>
	<summary>Tests dbt</summary>
	✅ Tests sur les sources RAW

Les tests unitaires vérifient la qualité des données brutes :

unicité (unique)

non-nullité (not_null)

intégrité référentielle (relationships)

valeurs autorisées (accepted_values)

🧩 Tests sur les modèles CURATION

Des tests de validation assurent la cohérence des données transformées :
```
version: 2
models:
  - name: curation_listings
    description: "Table nettoyée et enrichie des annonces Airbnb"
    columns:
      - name: price
        description: "Prix par nuit en euros"
        tests:
          - not_null

```
Tests unitaires personnalisés
Les unit tests vérifient la logique de transformation à partir d’exemples isolés.
```
unit_tests:
  - name: test_is_curation_listings_price_transformation_correct
    description: "Vérifie la transformation de la colonne price"
    model: curation_listings
    given:
      - input: ref('listings_snapshot')
        rows:
          - {price: '52.23$', DBT_VALID_TO: null}
          - {price: '$52.23', DBT_VALID_TO: null}
    expect:
      rows:
        - {price: 52.23}

```
</details>	
