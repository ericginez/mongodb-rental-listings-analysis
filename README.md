# Analyse de données locatives avec MongoDB, Polars et Power BI

Projet réalisé dans le cadre du parcours **Data Engineer OpenClassrooms**.

L’objectif est de concevoir et analyser une base de données NoSQL contenant
des annonces de locations de courte durée, puis de produire des indicateurs
métier exploitables dans Power BI.

Le projet utilise principalement :

- **MongoDB** pour le stockage documentaire ;
- **PyMongo** pour interroger la base ;
- **Polars** pour les traitements analytiques ;
- **Jupyter Notebook** pour documenter l’analyse ;
- **Power BI** pour la restitution visuelle.

---

## Contexte

Les données décrivent des annonces de locations de courte durée pour Paris
et Lyon.

Chaque annonce contient de nombreux attributs relatifs :

- au logement ;
- à l’hôte ;
- à sa localisation ;
- à sa capacité ;
- à son prix ;
- à sa disponibilité ;
- aux avis reçus ;
- aux conditions de réservation.

Le modèle documentaire MongoDB est adapté à ces données, car les annonces
comportent un grand nombre de champs hétérogènes, dont certains peuvent être
absents ou évoluer selon les extractions.

---

## Objectifs

Le projet répond aux objectifs suivants :

- stocker des annonces dans une base NoSQL MongoDB ;
- travailler avec un volume important de données semi-structurées ;
- sélectionner uniquement les champs nécessaires à une analyse ;
- convertir les documents MongoDB en DataFrame Polars ;
- nettoyer et typer les données ;
- construire des indicateurs sur les logements et les hôtes ;
- analyser la répartition géographique des annonces ;
- comparer les catégories de logements ;
- préparer une restitution dans Power BI ;
- expérimenter différentes architectures MongoDB locales.

---

## Technologies utilisées

| Technologie | Utilisation |
|---|---|
| MongoDB | Stockage documentaire des annonces |
| PyMongo | Connexion à MongoDB et extraction des documents |
| Polars | Nettoyage, typage et agrégations analytiques |
| Python 3.14 | Exécution du notebook |
| Jupyter Notebook | Analyse interactive et documentation |
| Power BI | Visualisation et restitution |
| Poetry | Gestion des dépendances Python |
| Git LFS | Versionnement du rapport Power BI |
| Git et GitHub | Versionnement du projet |

---

## Arborescence du dépôt

```text
.
├── .gitattributes
├── .gitignore
├── README.md
├── Data Dictionary.csv
├── Data Dictionary.xlsx
├── Projet 07.pbix
├── Projet_07.ipynb
├── poetry.lock
└── pyproject.toml
```

Les fichiers suivants ne sont pas publiés :

```text
listings_Paris.csv
listings_Lyon.csv
mongo-rs/
mongo-shard/
.ipynb_checkpoints/
```

Les fichiers CSV sources et les fichiers internes de MongoDB peuvent être
volumineux et sont reproductibles à partir des sources originales.

---

## Dépendances Python

Les dépendances sont déclarées dans `pyproject.toml`.

```toml
requires-python = ">=3.14"

dependencies = [
    "pymongo (>=4.16.0,<5.0.0)",
    "polars (>=1.39.0,<2.0.0)",
    "jupyter (>=1.1.1,<2.0.0)",
    "ipykernel (>=7.2.0,<8.0.0)"
]
```

Installation :

```powershell
poetry install
```

Lancement du notebook :

```powershell
poetry run jupyter notebook Projet_07.ipynb
```

---

## Sources de données

Le projet utilise deux fichiers d’annonces :

```text
listings_Paris.csv
listings_Lyon.csv
```

Le fichier parisien représente environ 185 Mo, ce qui dépasse la limite
classique d’un fichier GitHub.

Les données sources ne sont donc pas intégrées au dépôt.

Le dictionnaire de données est fourni dans deux formats :

```text
Data Dictionary.csv
Data Dictionary.xlsx
```

---

## Structure des données

Une annonce comporte notamment les familles d’informations suivantes.

### Identification

- `id`
- `listing_url`
- `name`
- `description`

### Hôte

- `host_id`
- `host_name`
- `host_since`
- `host_location`
- `host_is_superhost`
- `host_identity_verified`

### Localisation

- `neighbourhood`
- `neighbourhood_cleansed`
- `latitude`
- `longitude`

### Logement

- `property_type`
- `room_type`
- `accommodates`
- `bathrooms`
- `bedrooms`
- `beds`
- `amenities`

### Réservation et disponibilité

- `price`
- `minimum_nights`
- `maximum_nights`
- `availability_30`
- `availability_60`
- `availability_90`
- `availability_365`
- `instant_bookable`

### Avis

- `number_of_reviews`
- `number_of_reviews_ltm`
- `number_of_reviews_l30d`
- `review_scores_rating`
- `reviews_per_month`

---

## Base MongoDB analysée

Le notebook se connecte à une instance MongoDB locale :

```python
client = MongoClient("mongodb://localhost:27017/")
```

La base et la collection utilisées sont :

```text
Base       : locparis
Collection : listings
```

Le chargement initial des CSV dans MongoDB a été réalisé en amont du
notebook.

Le dépôt ne contient pas encore de script automatisant cet import.

---

## Projection MongoDB

L’analyse ne récupère pas tous les champs des documents.

Une projection PyMongo limite l’extraction aux attributs nécessaires :

```python
projection = {
    "_id": 0,
    "host_is_superhost": 1,
    "room_type": 1,
    "neighbourhood_cleansed": 1,
    "number_of_reviews": 1,
    "availability_365": 1,
}
```

Cette projection réduit :

- le volume transféré depuis MongoDB ;
- la mémoire consommée ;
- le temps de création du DataFrame ;
- le nombre de colonnes à nettoyer.

---

## Chargement dans Polars

Les documents MongoDB sont convertis en DataFrame Polars :

```python
docs = list(collection.find({}, projection))
df = pl.DataFrame(docs)
```

Les types sont ensuite normalisés :

```python
df = df.with_columns([
    pl.col("host_is_superhost").cast(pl.Utf8, strict=False),
    pl.col("room_type").cast(pl.Utf8, strict=False),
    pl.col("neighbourhood_cleansed").cast(pl.Utf8, strict=False),
    pl.col("number_of_reviews").cast(pl.Int64, strict=False),
    pl.col("availability_365").cast(pl.Int64, strict=False),
])
```

L’analyse porte sur **95 885 annonces parisiennes**.

---

## Indicateurs calculés

### Nombre estimé de jours réservés par mois

Le notebook utilise la disponibilité annuelle pour estimer le nombre de jours
non disponibles :

```text
jours non disponibles = 365 - availability_365
```

L’estimation mensuelle est :

```text
jours non disponibles estimés par mois
    = (365 - availability_365) / 12
```

Cette mesure constitue une approximation.

Un logement peut être indisponible parce qu’il est réservé, mais également
parce que l’hôte l’a bloqué. Elle ne correspond donc pas nécessairement à un
nombre réel de réservations.

Résultats moyens par type de logement :

| Type de logement | Jours non disponibles estimés par mois |
|---|---:|
| Private room | 20,93 |
| Entire home/apt | 19,74 |
| Shared room | 19,26 |
| Hotel room | 15,93 |

---

### Médiane du nombre d’avis

La médiane globale est :

```text
3 avis
```

La médiane est utilisée plutôt que la moyenne afin de limiter l’influence des
logements possédant un nombre exceptionnellement élevé d’avis.

---

### Analyse des super-hôtes

Répartition des annonces :

| Statut | Nombre d’annonces |
|---|---:|
| Non super-hôte | 81 035 |
| Super-hôte | 14 776 |
| Valeur vide | 74 |

Médiane du nombre d’avis :

| Catégorie | Médiane |
|---|---:|
| Super-hôte | 24 |
| Non super-hôte | 2 |

Les annonces des super-hôtes possèdent donc une médiane d’avis nettement
supérieure.

Cette différence ne démontre cependant pas à elle seule une relation de
causalité.

---

### Densité des logements par quartier

Les annonces sont regroupées par `neighbourhood_cleansed`.

Les quartiers possédant le plus d’annonces sont :

| Quartier | Nombre de logements | Part |
|---|---:|---:|
| Buttes-Montmartre | 10 555 | 11,01 % |
| Popincourt | 8 430 | 8,79 % |
| Vaugirard | 7 802 | 8,14 % |
| Batignolles-Monceau | 6 857 | 7,15 % |
| Entrepôt | 6 558 | 6,84 % |

Buttes-Montmartre représente à lui seul environ 11 % des annonces analysées.

---

### Quartiers les moins disponibles

Les quartiers présentant le plus grand nombre moyen de jours non disponibles
par mois sont :

| Quartier | Jours non disponibles estimés | Logements |
|---|---:|---:|
| Ménilmontant | 21,62 | 5 271 |
| Buttes-Chaumont | 21,21 | 5 465 |
| Buttes-Montmartre | 21,06 | 10 555 |
| Entrepôt | 20,99 | 6 558 |
| Popincourt | 20,95 | 8 430 |
| Gobelins | 20,76 | 3 304 |
| Reuilly | 20,61 | 4 003 |
| Panthéon | 20,18 | 3 010 |
| Vaugirard | 20,05 | 7 802 |
| Observatoire | 19,82 | 3 611 |

---

## Analyse Power BI

Le fichier :

```text
Projet 07.pbix
```

contient le rapport Power BI associé au projet.

Il permet de restituer graphiquement les indicateurs issus de l’analyse,
notamment :

- la répartition des annonces par quartier ;
- la distribution par type de logement ;
- la comparaison entre super-hôtes et autres hôtes ;
- la disponibilité des logements ;
- les principaux indicateurs sur les avis.

Le fichier PBIX est versionné avec Git LFS afin de ne pas alourdir directement
l’historique Git.

---

## Architectures MongoDB locales

Les dossiers locaux montrent que plusieurs configurations MongoDB ont été
expérimentées :

```text
mongo-rs/
mongo-shard/
```

Ils contiennent notamment :

- des fichiers WiredTiger ;
- des journaux ;
- des index ;
- des métadonnées de configuration ;
- des données de replica set ;
- des données associées à une architecture shardée.

Ces répertoires représentent l’état interne de MongoDB et ne constituent pas
des fichiers de configuration réutilisables.

Ils sont donc exclus du dépôt.

Le notebook publié utilise une connexion directe sur :

```text
mongodb://localhost:27017/
```

Le dépôt ne fournit actuellement ni fichier Docker Compose ni script permettant
de reconstruire automatiquement le replica set ou le cluster shardé.

---

## Sécurité et gestion des fichiers

Le `.gitignore` exclut :

- les volumes MongoDB ;
- les fichiers WiredTiger ;
- les journaux MongoDB ;
- les CSV de données ;
- les checkpoints Jupyter ;
- les environnements virtuels ;
- les fichiers temporaires ;
- les secrets éventuels.

Le rapport Power BI est suivi avec la règle Git LFS :

```gitattributes
*.pbix filter=lfs diff=lfs merge=lfs -text
```

---

## Limites

Plusieurs limites doivent être prises en compte :

- les données sources ne sont pas incluses dans le dépôt ;
- l’import CSV vers MongoDB n’est pas automatisé ;
- l’analyse reproductible publiée concerne Paris ;
- le fichier Lyon n’est pas exploité dans le notebook final ;
- les architectures replica set et shardée ne sont pas configurables depuis
  les fichiers versionnés ;
- l’indicateur dérivé de `availability_365` est une estimation ;
- une indisponibilité ne correspond pas nécessairement à une réservation ;
- le rapport Power BI dépend de données locales non publiées.

---

## Évolutions possibles

Le projet pourrait être enrichi avec :

- un script automatisé d’import des CSV ;
- un fichier Docker Compose pour MongoDB ;
- une configuration reproductible du replica set ;
- une configuration reproductible du sharding ;
- une analyse comparative Paris-Lyon ;
- des pipelines d’agrégation exécutés directement dans MongoDB ;
- la création et l’analyse d’index ;
- des tests automatisés de qualité ;
- un export Parquet destiné à Power BI ;
- un jeu de données d’exemple réduit ;
- une documentation du modèle documentaire ;
- une chaîne d’exécution complète.

---

## Compétences démontrées

- conception d’une base documentaire ;
- manipulation de données semi-structurées ;
- interrogation de MongoDB avec PyMongo ;
- utilisation de projections MongoDB ;
- traitement analytique avec Polars ;
- nettoyage et typage de données ;
- agrégation par catégories ;
- calcul d’indicateurs métier ;
- analyse géographique ;
- interprétation critique des résultats ;
- création d’un rapport Power BI ;
- gestion de fichiers volumineux avec Git LFS ;
- documentation des limites de reproductibilité.

---

## Auteur

**Eric Ginez**  
Parcours Data Engineer — OpenClassrooms