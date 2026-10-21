# Phase 1 — Cadrage et recueil des besoins

## 1.1 Objectif de la phase

Avant toute modélisation de données, cette phase vise à traduire les compétences à démontrer en exigences fonctionnelles précises et traçables jusqu'aux tables et colonnes qui les serviront. Aucun élément du schéma de données (Phase 2) c'est retenu s'il ne répond pas à une exigence formulée ici.

## 1.2 Cadre du projet

| Paramètre | Valeur |
|---|---|
| Objet | Système de suivi et de contrôle qualité d'une enquête administrative |
| Nature | Méthodologie inspirée de pratiques professionnelles d'enquête statistique |
| Données | Anonymisées 
| Population couverte | 2 068 unités administratives (CT / EPA / SDL), 12 régions |
| Type d'opération | Recensement exhaustif — aucune pondération statistique |
| Durée de référence | 7 mois (01/01/2025 → 01/07/2025) |

## 1.3 Exigences techniques et fonctionnelles

| Réf. | Exigence | Type | Justification |
|---|---|---|---|
| EX-01 | Modéliser un schéma en étoile, compatible SQLite et Azure SQL | Technique | Data modeling |
| EX-02 | Calculer le taux de réponse par région | Fonctionnelle | SQL & analyse statistique |
| EX-03 | Calculer le taux d'imputation par région | Fonctionnelle | SQL & analyse statistique |
| EX-04 | Détecter les valeurs aberrantes selon deux méthodes distinctes (IQR et Z-score) | Fonctionnelle | SQL & analyse statistique |
| EX-05 | Classer (ranking) les régions selon la qualité des données | Fonctionnelle | SQL & analyse statistique |
| EX-06 | Suivre l'évolution du taux de réponse dans le temps | Fonctionnelle | SQL & analyse statistique |
| EX-07 | Construire un indice composite de qualité (taux de réponse, taux de retard, taux d'anomalie) | Technique | SQL & analyse statistique |
| EX-08 | Restituer une carte régionale interactive | Technique | Dashboard Power BI |
| EX-09 | Afficher des KPIs de taux de réponse et d'imputation | Fonctionnelle | Dashboard Power BI |
| EX-10 | Générer des alertes visuelles conditionnelles pour les régions à risque | Fonctionnelle | Dashboard Power BI |
| EX-11 | Suivre les tendances temporelles via l'intelligence temporelle DAX | Fonctionnelle | Dashboard Power BI |
| EX-12 | Garantir une architecture portable vers Azure SQL Database | Technique | Data modeling |

## 1.4 Matrice de traçabilité

| Réf. | Donnée requise | Source | Calcul |
|---|---|---|---|
| EX-02 | Unités avec `non_reponse` = Non / total unités, par région | `Fait_Qualite` ⨝ `Dim_Unite_Enquetee` ⨝ `Dim_Region` | Agrégat SQL |
| EX-03 | Unités avec `methode_imputation` renseignée / total unités ayant répondu, par région | `Fait_Qualite.methode_imputation` | Agrégat SQL |
| EX-04 | Bornes IQR et Z-score sur `ecart_gps_metres` (Fait_Suivi_Terrain) et `nb_erreurs_capi` (Fait_Qualite), par strate région × type d'unité | `Fait_Suivi_Terrain`, `Fait_Qualite` | SQL + Python (scipy) |
| EX-05 | Rang de chaque région sur l'indice composite | Vue SQL sur l'indice composite | `RANK()` |
| EX-06 | Évolution des indicateurs par trimestre | `Fait_Suivi_Terrain.date_id` / `Fait_Qualite.date_id` ⨝ `Dim_Calendrier` | Série temporelle SQL/DAX |
| EX-07 | Indice = f(taux de réponse, taux de retard, taux d'anomalie) — taux de retard = % unités avec `nb_tentatives` ≥ 3 indicateur proxy | `Fait_Qualite`, `Fait_Suivi_Terrain` | Formule pondérée (Phase 4) |
| EX-08–11 | Restitution des indicateurs ci-dessus | Power BI sur SQLite/Azure SQL | DAX + Power Query |

## 1.5 Décisions de cadrage actées

| Décision | Choix retenu | Raison |
|---|---|---|
| Schéma de référence | Schéma réel validé (Dim_Superviseur, Dim_Region, Dim_Type_Unite, Dim_Enqueteur, Dim_Unite_Enquetee, Dim_Calendrier, Fait_Allocation_Ressources, Fait_Suivi_Terrain, Fait_Qualite) | Remplace les hypothèses initiales (`Fait_Saisie` n'existe pas) |
| Lien temporel sur les faits | Ajout d'une clé étrangère `date_id` sur `Fait_Suivi_Terrain` et `Fait_Qualite` | Nécessaire pour EX-06 (tendance dans le temps), absente du schéma initial |
| Variable de détection d'anomalies | `ecart_gps_metres` **et** `nb_erreurs_capi`, en parallèle | Pas de variable de montant disponible — le système surveille la qualité opérationnelle, pas les données d'investissement |
| Méthodes d'outliers | IQR **et** Z-score, en parallèle, par strate région × type d'unité | Exigence EX-04 explicite |
| Définition du taux de retard | % d'unités avec `nb_tentatives` ≥ 3 | Indicateur proxy car Aucune paire de dates (assignation/fin) disponible pour un calcul de retard réel ; seuil ajustable en Phase 4 |
| Indice composite | 3 composantes : réponse, retard (proxy tentatives), anomalie | Aligné strictement sur EX-07 |
| Calendrier ouvrable | Week-ends exclus uniquement (version simplifiée) | Raffinement possible ultérieurement : exclusion des jours fériés marocains |
