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

# Résultats de nos Analyses SQL

## 🏙 Quelle est la distribution des prix par quartier ?

```
WITH categorisation_price as (
    SELECT 
        p.host_id,
        CASE 
            WHEN price < 100 THEN 'Budget'
            WHEN price BETWEEN 100 AND 199 THEN 'Standard'
            WHEN price BETWEEN 200 AND 399 THEN 'Confort'
            WHEN price BETWEEN 400 AND 699 THEN 'Premium'
            WHEN price BETWEEN 700 AND 999 THEN 'Luxe'
            WHEN price >= 1000 THEN 'Exceptionnel'
            ELSE 'Inconnu'
        END AS price_category,
        h.host_neighbourhood
    FROM
        {{ ref("curation_listings") }} p
    INNER JOIN {{ ref("curation_hosts") }} h
    ON p.host_id = h.host_id
)

SELECT
    host_neighbourhood,
    price_category,
    count(price_category)
FROM categorisation_price 
GROUP BY 1, 2
ORDER BY 1, 2, 3 DESC
```
L’analyse montre que la majorité des logements sont de catégorie Standard ou Confort (environ 70 %), représentant une offre de milieu de gamme.
Les logements Budget (15–20 %) se trouvent surtout dans les quartiers périphériques comme Bos en Lommer ou Oost, tandis que les offres Premium et Luxe (environ 10 %) se concentrent dans les zones centrales et touristiques telles que Grachtengordel, De Pijp ou Jordaan.

Les quartiers les plus dynamiques sont Oud-West, Grachtengordel, De Pijp et Jordaan, qui regroupent la majorité des annonces.
En résumé, plus on s’éloigne du centre, plus les prix ont tendance à diminuer.

=> Les résultats ont été exportés au format CSV

## ⭐ Comment se répartissent les super-hôtes dans la ville ?

```
WITH distribution_hosts as (
    SELECT
        host_neighbourhood,
        count(host_id) nb_host
    from
        AIRBNB_PROJECT.curation.curation_hosts
    where is_superhost = TRUE
    GROUP BY 1
)


SELECT
    *
FROM    
    distribution_hosts
ORDER BY nb_host DESC
```
L’analyse montre que les super hôtes sont principalement concentrés dans les quartiers centraux d’Amsterdam.
Les zones les plus représentées sont Oud-West (78 super hôtes), Grachtengordel (63), Jordaan (42) et De Pijp (41), qui regroupent à eux seuls la majorité des super hôtes.

Des quartiers comme Nieuwmarkt en Lastage, Westelijke Eilanden et Oosterparkbuurt affichent également une présence notable.
À l’inverse, les quartiers périphériques comptent très peu de super hôtes (souvent moins de 5).

En résumé, les super hôtes se concentrent dans les zones touristiques et centrales, là où la demande et la qualité de service sont les plus élevées.
<img width="1183" height="546" alt="image" src="https://github.com/user-attachments/assets/913169d6-1ff5-4c90-a7d6-ff761130dc0c" />

## 💰 Existe-t-il une corrélation entre le statut de super-hôte et le prix moyen des annonces ?

```
WITH super_host as (
    SELECT 
        ROUND(AVG(l.price),2) avg_price,
        h.host_neighbourhood quartier,
        'super host'
    FROM AIRBNB_PROJECT.curation_info.curation_listings l
    JOIN AIRBNB_PROJECT.curation.curation_hosts h
    ON l.host_id = h.host_id
    WHERE h.is_superhost = TRUE
    GROUP BY quartier
), no_super_host as (
    SELECT 
        ROUND(AVG(l.price),2) avg_price,
        h.host_neighbourhood quartier,
        'non super host'
    FROM AIRBNB_PROJECT.curation_info.curation_listings l
    JOIN AIRBNB_PROJECT.curation.curation_hosts h
    ON l.host_id = h.host_id
    WHERE h.is_superhost = FALSE
    GROUP BY quartier
)

SELECT *
FROM super_host
UNION
SELECT *
FROM no_super_host
ORDER BY quartier
```
L’analyse montre qu’il n’y a pas de lien direct entre le statut de super hôte et un prix plus élevé.

Au contraire, les super hôtes pratiquent en moyenne des tarifs légèrement inférieurs à ceux des non super hôtes (environ 180 € contre 220 €).
Cela suggère que le statut de super hôte reflète davantage la qualité du service et la fiabilité, plutôt qu’un positionnement haut de gamme.

=> Les résultats ont été exportés au format CSV

##  tendances touristiques
```
with review_per_year as (
    SELECT
        YEAR(review_date) year,
        count(listing_id)total_review
    FROM
        AIRBNB_PROJECT.curation.curation_reviews
    GROUP BY year
), sejours_estimes as (
    SELECt
        year,
        total_review,
        ROUND(total_review::float/0.8, 0) sejour_estimes
    FROM
        review_per_year
),tourisme_airbnb as (
    select
        year(t.year) year,
        t.tourists tourist_total,
        s.total_review,
        s.sejour_estimes,
        (s.sejour_estimes::float/t.tourists) * 100 pct_tourists_airbnb,
        t.tourists - s.sejour_estimes nb_touristes_hotels
    from
        AIRBNB_PROJECT.curation.curation_tourists_per_year t
    LEFT JOIN sejours_estimes s
    ON year(t.year) = s.year
)

select 
    year,
    tourist_total,
    total_review,
    sejour_estimes,
    pct_tourists_airbnb,
    nb_touristes_hotels,
    LAG(pct_tourists_airbnb) OVER(ORDER BY year) as precedent_year,
    pct_tourists_airbnb -  LAG(pct_tourists_airbnb) OVER(ORDER BY year) evolution_tourist_airbnb,
    round(
    ((pct_tourists_airbnb / nullif(lag(pct_tourists_airbnb) over (order by year), 0)) - 1) * 100,2) croissance_pct
    
from tourisme_airbnb
```

Entre 2012 et 2023, la part des touristes utilisant Airbnb a connu une croissance spectaculaire, passant de 0,01 % à plus de 1 % du total des visiteurs.
Cette évolution traduit une montée en puissance rapide d’Airbnb, surtout entre 2015 et 2021, avant une stabilisation récente.

Si les hôtels restent largement majoritaires, les touristes privilégient de plus en plus les hébergements Airbnb, perçus comme plus flexibles et adaptés aux séjours individuels ou de longue durée.
<img width="1979" height="1180" alt="image" src="https://github.com/user-attachments/assets/8f5e2a32-56db-4229-947e-005d89cc9677" />
