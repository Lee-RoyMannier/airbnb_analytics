# Le jeu de données Airbnb
## Source: 
Le jeu de données a été téléchargé depuis le site https://insideairbnb.com/get-the-data/ qui regroupe les données Airbnb 
pour plusieurs villes. Pour notre travail, nous avons choisi la ville d'Amsterdam correspondant à un extrait du 11 Mars 2024

<details>
   <summary>Mise en place de dbt (sur dbt cloud) et de snowflake</summary>

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

# Chapitre 1: Mise en place de l'environnement
## Configuration de Snowflake
Pour configurer l'accès de DBT à Snowflake, copiez-coller cet ensemble de requêtes SQL dans Snowflake et exécutez-le. 
N'hésitez pas à changer le mot de passe pour plus de sécurité.  
  Le fichier de détail des commandes pour la création du rôle et la mise en place de dbt ce trouve dans le dépôt git sous le nom "CREATION ET MISE EN PLACE DE DBT SUR SNOWFLAKE"
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

## Chargement des donneées
Pour charger les données dans Snowflake, il nous faut faire un peu de gymnastique SQL.
cet ensemble de requêtes et de les exécuter dans Snowflake. 
   Le fichier de détail des commandes pour la création des tables dans snowflake ce trouve dans le dépôt git sous le nom "CREATION TABLE SUR SNOWFLAKE"
```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE AIRBNB_PROJECT;
USE SCHEMA RAW;    
    
create or replace api integration integration_jeu_de_donnees_github
api_provider = git_https_api
api_allowed_prefixes = ('https://github.com/QuantikDataStudio')
enabled = true;

create or replace git repository jeu_de_donnees_airbnb
api_integration = integration_jeu_de_donnees_github
origin = 'https://github.com/QuantikDataStudio/dbt.git';

create or replace file format format_jeu_de_donnees
type = csv
skip_header = 1
field_optionally_enclosed_by = '"';

CREATE TABLE AIRBNB_PROJECT.RAW.HOSTS
(
    host_id                STRING,
    host_name              STRING,
    host_since             DATE,
    host_location          STRING,
    host_response_time     STRING,
    host_response_rate     STRING,
    host_is_superhost      STRING,
    host_neighbourhood     STRING,
    host_identity_verified STRING
);


insert INTO AIRBNB_PROJECT.RAW.HOSTS (SELECT $1 as host_id,
                                     $2 as host_name,
                                     $3 as host_since,
                                     $4 as host_location,
                                     $5 as host_response_time,
                                     $6 as host_response_rate,
                                     $7 as host_is_superhost,
                                     $8 as host_neighbourhood,
                                     $9 as host_identity_verified
                              from @jeu_de_donnees_airbnb/branches/main/dataset/hosts.csv
                          (FILE_FORMAT => 'format_jeu_de_donnees'));

CREATE TABLE AIRBNB_PROJECT.RAW.LISTINGS
(
    id                     STRING,
    listing_url            STRING,
    name                   STRING,
    description            STRING,
    neighbourhood_overview STRING,
    host_id                STRING,
    latitude               STRING,
    longitude              STRING,
    property_type          STRING,
    room_type              STRING,
    accommodates           integer,
    bathrooms              FLOAT,
    bedrooms               FLOAT,
    beds                   FLOAT,
    amenities              STRING,
    price                  STRING,
    minimum_nights         INTEGER,
    maximum_nights         INTEGER
);

INSERT INTO AIRBNB_PROJECT.RAW.LISTINGS (SELECT $1  AS id,
                                        $2  AS listing_url,
                                        $3  AS name,
                                        $4  AS description,
                                        $5  AS neighbourhood_overview,
                                        $6  AS host_id,
                                        $7  AS latitude,
                                        $8  AS longitude,
                                        $9  AS property_type,
                                        $10 AS room_type,
                                        $11 AS accommodates,
                                        $12 AS bathrooms,
                                        $13 AS bedrooms,
                                        $14 AS beds,
                                        $15 AS amenities,
                                        $16 AS price,
                                        $17 AS minimum_nights,
                                        $18 AS maximum_nights
                                 from @jeu_de_donnees_airbnb/branches/main/dataset/listings.csv
                          (FILE_FORMAT => 'format_jeu_de_donnees'));


CREATE TABLE AIRBNB_PROJECT.RAW.REVIEWS
(
    listing_id  STRING,
    date        DATE
);

INSERT INTO AIRBNB_PROJECT.RAW.REVIEWS (SELECT $1 as listing_id,
                                       $2 as date
                                from @jeu_de_donnees_airbnb/branches/main/dataset/reviews.csv
                                    (FILE_FORMAT => 'format_jeu_de_donnees'));
```
Ne pas oublier de rentrer toutes les informations importante (comme le rôle, la connection ect) dans les paramètre du projet DBT
</details>

<details>
   <summary>Création de nos modèles dans DBT</summary>
   Si je devais expliquer ce qu'est un modèles dbt, je dirai:
   Les modèles DBT sont des fichiers qui contiennent des requêtes SQL qui servent à définir des tables, vue ou de simples parties d'une requête plus large, et qui permettent ensuite de travailler avec
   Le SQL pour `curation_hosts` est le suivant:

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

Le SQL pour `curation_listings` est le suivant:
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

maintenant que c'est fait, nous allons pouvoir mettre un vrai nom à notre projet dans "dbt_project.yaml"
<img width="623" height="137" alt="image" src="https://github.com/user-attachments/assets/78239081-08d3-46cb-a5b5-ba07b5e5a839" />
<img width="385" height="122" alt="image" src="https://github.com/user-attachments/assets/262872d2-567f-4721-a1ee-c23f3769b807" />



En rajoutant +schema:curation, je vais créer un schema dans snowflake me permettant de séparer mes tables curation du raw et ainsi garder une certaine cohérence.
</details>
