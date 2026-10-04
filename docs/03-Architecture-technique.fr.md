# Phase 3 — Architecture technique et infrastructure

## 3.1 Objectif de la phase

Définir l'environnement technique sur lequel le schéma de la Phase 2 sera implémenté : moteur de base de données, outil de génération des données simulées, et chaîne d'analyse — en garantissant une portabilité réelle vers Azure SQL Database (EX-01, EX-12), pas seulement déclarative.

## 3.2 Composants retenus

| Composant | Outil retenu | Justification |
|---|---|---|
| Base de données (portfolio) | SQLite | Aucune installation serveur, fichier `.db` versionnable dans le repo, suffisant pour 2 068 lignes |
| Compatibilité cible | Azure SQL Database | Exigée par EX-12 — implique d'éviter les fonctions strictement propriétaires à SQLite dans les requêtes destinées à être réutilisées |
| Génération des données simulées | Python (`faker`, `random`, `numpy`) | Contrôle fin des distributions et des taux d'anomalies/non-réponse |
| Chargement des données dans SQLite | Python (`sqlite3` natif) | Un script `generate_data.py` peuple les 9 tables dans l'ordre des dépendances (dimensions avant faits) |
| Analyse (IQR/Z-score, indice composite) | SQL + Python (`pandas`, `scipy`) | SQL pour les agrégats simples, Python pour les calculs statistiques (EX-04, EX-07) |
| Restitution | Power BI Desktop | Connexion directe au fichier SQLite (ou export CSV intermédiaire) |

## 3.3 Décisions techniques actées

| Décision | Choix retenu | Raison |
|---|---|---|
| Ordre de génération des données | Dimensions d'abord (superviseur → région → type unité → enquêteur → unité enquêtée → calendrier), puis faits | Respect des contraintes FK |
| Respect du 1:1 à la génération | Une seule ligne par `unite_id` dans `Fait_Suivi_Terrain` et `Fait_Qualite` | Cohérence avec la contrainte UNIQUE actée en Phase 2 |
| Ancrage enquêteur ↔ région à la génération | Chaque enquêteur est rattaché à une seule région dès sa création ; les unités piochent leur agent uniquement parmi ceux de leur propre région | Empêche qu'un enquêteur se retrouve avec des unités de plusieurs régions différentes |
| Source des effectifs par région | Tableau EIAP réel (besoins en moyens humains et matériels), codé en dur dans `REGIONS_DATA` | Remplace une répartition uniforme par les chiffres exacts par région (ex. Marrakech-Safi : 299 unités/7 enquêteurs/7 véhicules/8 tablettes ; Eddakhla-Oued Eddahab : 24 unités/1 enquêteur/1 véhicule/1 tablette) |
| Nombre de superviseurs | 12 au total (1 par région, strict 1:1) | Chaque superviseur est affecté à sa propre région où il réside |
| Taux d'anomalie simulé | **~10–15%** des unités avec `ecart_gps_metres` aberrant ou `nb_erreurs_capi` élevé | Modéré : suffisant pour démontrer la détection IQR/Z-score (EX-04) sans fausser excessivement les KPIs globaux |
| Taux de non-réponse simulé | **~10%** des unités avec `non_reponse` = Oui | Réaliste pour une enquête administrative obligatoire, sans dominer les indicateurs |
| Reproductibilité | `random.seed(42)` fixé dans le script | Garantit que chaque exécution produit exactement les mêmes données, pour des analyses comparables |
| Emplacement du fichier base | `data/enquete_administrative.db` à la racine du repo | Cohérent avec la structure GitHub déjà posée |
| Emplacement du script de génération | `scripts/generate_data.py` | Séparation claire code / données |

## 3.4 Livrable de fin de phase
