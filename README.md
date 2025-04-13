# Pipeline de Données Agricoles

Ce projet fournit un pipeline robuste de data engineering pour le traitement de données agricoles à l’aide d’**Azure Data Factory**, **Azure Databricks** et **Azure Synapse Analytics**.  
Le pipeline automatise l’ingestion, le traitement et le stockage des données dans les couches **bronze**, **silver** et **gold**, préparant les données pour l’analyse et la visualisation.

---

## Cas d’Usage

Les données agricoles (météo, sol, production) sont essentielles pour améliorer les rendements, anticiper les conditions climatiques et gérer les ressources de manière durable.  
Les parties prenantes telles que les agriculteurs, les ingénieurs agronomes, les instituts de recherche ou les décideurs politiques s’appuient sur des informations précises et à jour pour :

- Planifier les cultures et les ressources.
- Évaluer les risques liés au climat.
- Prendre des décisions éclairées basées sur les données.

En automatisant la collecte et la transformation des données agricoles, ce pipeline permet aux utilisateurs de **prédire, analyser et optimiser** les activités agricoles avec précision et rapidité.

## Architecture & Technologies

Ce projet repose sur une architecture moderne de type **Medallion** (Bronze / Silver / Gold) pour le traitement et l’analyse des données météorologiques.

### Pipeline global

Le pipeline suit cette structure logique :

1. **Azure Data Factory (ADF)** : Orchestration de l’ingestion depuis un fichier CSV de villes et exécution de notebooks.
2. **Azure Data Lake Storage Gen2** : Stockage des données brutes, semi-traitées et enrichies.
3. **Azure Databricks** : Traitement Spark distribué en 3 couches :
   - **Bronze** : Données brutes collectées via une API météo.
   - **Silver** : Données nettoyées, structurées (séparation date/heure, format parquet).
   - **Gold** : Données enrichies (ville, pays, classification de température).
4. **Azure Synapse Analytics** : Lecture et requêtage des données Parquet depuis le lac.
5. **Power BI** : Création de visualisations interactives à partir des données analysées.

### Architecture visuelle

![Architecture Pipeline](./agri-pipline.png)

### Stack technique utilisée

| Composant             | Rôle principal                          |
|-----------------------|------------------------------------------|
| Azure Data Factory    | Orchestration des notebooks              |
| Azure Storage Gen2    | Stockage structuré en Bronze/Silver/Gold |
| Azure Databricks      | Traitement Spark, API, enrichissement    |
| Azure Synapse         | Requêtes SQL sur Parquet                 |
| Power BI              | Visualisation des données finales        |
| API OpenMeteo         | Source de données météo                  |
| Python + PySpark      | Traitement de données                    |
| Librairie `reverse_geocoder` | Récupération ville/pays à partir des coordonnées |

---


## Azure Setup

Cette section décrit la mise en place des services Azure nécessaires au fonctionnement du pipeline. L’objectif est d’assurer une communication fluide entre les différentes briques du projet.

### 1. Création des ressources Azure

Plusieurs services Azure ont été provisionnés pour mettre en œuvre l’architecture Medallion :

- **Azure Data Lake Storage Gen2**  
  Utilisé pour stocker les fichiers dans les différentes couches : `bronze`, `silver` et `gold`.  
  → Trois conteneurs ont été créés : `bronze`, `silver` et `gold`.

- **Azure Databricks**  
  Un workspace Databricks a été créé pour exécuter les traitements distribués avec PySpark.  
  Un cluster a été lancé, avec la librairie `reverse_geocoder` installée pour enrichir les données par géolocalisation.

- **Azure Synapse Analytics**  
  Créé pour interroger les fichiers Parquet présents dans la couche Gold, via le langage SQL.

- **Azure Data Factory (ADF)**  
  Utilisé pour orchestrer l’ensemble du pipeline : déclenchement automatique chaque jour à minuit, itération sur les villes, exécution des notebooks.

---

### 2. Connexion entre services

Afin que tous les services puissent interagir avec le Data Lake, plusieurs étapes de configuration ont été nécessaires :

- **Création des credentials + external location dans Databricks**  
  Ces éléments permettent à Databricks d’accéder aux fichiers dans le Data Lake via les chemins `abfss://`.

- **Gestion des rôles et permissions (IAM)**  
  Il a fallu attribuer le rôle suivant au service principal de Databricks :  
  → `Storage Blob Data Contributor`  
  Ceci a été fait au niveau du **container** et non uniquement au niveau du compte de stockage.

---
## Databricks – Traitement des données

Le traitement des données se fait en trois étapes à l’aide de notebooks PySpark dans Databricks. Chaque notebook correspond à une couche du pipeline : Bronze, Silver ou Gold.

---

### 1. Configuration du cluster et librairie

Un cluster Databricks a été créé avec un runtime compatible PySpark.  
Nous avons installé la librairie suivante dans le cluster :

- `reverse_geocoder` → permet d'enrichir chaque ligne avec le nom de la ville et le code pays à partir de la latitude et longitude.

---

### 2. Bronze Notebook – Récupération des données météo

#### Étapes du traitement :

- Lecture des paramètres transmis par Azure Data Factory : `latitude`, `longitude`, `ville`, `pays`, `today`
- Requête HTTP vers l’API [Open-Meteo](https://open-meteo.com/), avec récupération des paramètres :
  - `temperature_2m`
  - `soil_temperature_0cm`
  - `precipitation`
- Vérification du contenu retourné par l’API
- Ajout des champs `latitude`, `longitude`, `ville`, `pays`, `date` dans le JSON
- Sauvegarde d’un fichier JSON par ville et par jour dans le conteneur **bronze**, chemin : *abfss://bronze@agristorage2025.dfs.core.windows.net/meteo/<pays><ville><today>.json*

---

### 3. Silver Notebook – Transformation des données

Le notebook Silver lit les fichiers JSON du Bronze, nettoie et structure les données.

#### Traitements effectués :

- Extraction des listes `time`, `temperature_2m`, `soil_temperature_0cm`, `precipitation`
- Utilisation de `posexplode` pour aligner chaque valeur sur son horodatage
- Conversion du champ `time` en type `timestamp`
- Séparation de `datetime` en deux colonnes : `date` (YYYY-MM-DD) et `heure` (HH:mm)
- Ajout des colonnes `latitude`, `longitude`, `ville`, `pays`
- Enregistrement du résultat au format **Parquet** dans le conteneur Silver :

_abfss://silver@agristorage2025.dfs.core.windows.net/meteo/<pays><ville><today>.parquet_

---

### 4. Gold Notebook – Enrichissement et préparation finale

Le Gold Notebook lit tous les fichiers Silver du jour, enrichit les données et les agrège.

#### Étapes clés :

- Lecture de tous les fichiers du jour :

<pre> df = spark.read.parquet(f"{silver_adls}/meteo/*_{today}.parquet") </pre>

- Ajout des colonnes country_code et city à l’aide de la fonction reverse_geocoder

- Un UDF PySpark personnalisé a été utilisé pour intégrer reverse_geocoder dans le pipeline Spark.

- Ajout d'une colonne stemp_class qui catégorise les températures :
   - <10°C → froid
   - entre 10°C et 25°C → modéré
   - 25°C → chaud
   - Écriture dans le conteneur Gold :
