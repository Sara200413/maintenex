# Maintenex (GearFlow) - Module de Maintenance Industrielle (GMAO) pour ERP

[![Java](https://img.shields.io/badge/Java-21%20LTS-orange.svg)](https://www.oracle.com/java/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.1-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Angular](https://img.shields.io/badge/Angular-22-red.svg)](https://angular.dev/)
[![Oracle Database](https://img.shields.io/badge/Oracle-21c%20XE%20%2F%20Free-crimson.svg)](https://www.oracle.com/database/)
[![License](https://img.shields.io/badge/Architecture-Clean%20%2F%20N--Tiers-blue.svg)]()

---

## 1. Présentation du Projet et Problématique Métier

### 1.1 Contexte Industriel
Dans de nombreux environnements manufacturiers et sites de production, la gestion des équipements et des pannes repose encore sur des outils informels (fiches papier, tableurs Excel isolés, communications directes non archivées).

Cette dispersion engendre plusieurs risques critiques :
* **Manque de visibilité :** Impossibilité d'identifier immédiatement les machines en service, en panne ou en révision.
* **Perte de traçabilité :** Absence d'historique centralisé des pannes et des actions correctives menées.
* **Dérive des coûts opérationnels :** Difficulté à mesurer les dépenses financières réelles liées à la maintenance et à l'immobilisation des actifs.
* **Surcharge des équipes :** Affectation non maîtrisée des techniciens sans vision de leur disponibilité.

### 1.2 La Solution : Maintenex
**Maintenex** est conçu comme un **noyau fonctionnel de GMAO (Gestion de Maintenance Assistée par Ordinateur)** autonome, destiné à s'intégrer au sein d'un système ERP d'entreprise. Il permet de :
1. Centraliser et suivre le cycle de vie de chaque équipement industriel via un identifiant unique.
2. Planifier et exécuter les opérations de maintenance **préventive** (révisions systématiques pour éviter l'arrêt) et **corrective** (réparation consécutive à une défaillance).
3. Piloter l'affectation des techniciens de maintenance selon leur disponibilité1].
4. Alimenter un tableau de bord analytique fournissant les indicateurs de fiabilité clés : nombre de défaillances, coût cumulé et **MTTR** (*Mean Time To Repair*).

---

## 2. Architecture Métier & Modélisation du Domaine

Le cœur applicatif est articulé autour de **3 entités pivot** et de **3 énumérations d'état**, garantissant la rigueur du modèle conceptuel de données:

### 2.1 Les Entités Métier
* **`Equipement` :** Représente l'actif physique de l'usine (machine, compresseur, moteur).
    * Attributs : `id`, `code` (unique), `nom`, `type`, `localisation`, `dateMiseEnService`, `statut`.
    * Attribut dérivé `/nbPannes` : Calculé dynamiquement à partir de la somme de ses interventions de type `CORRECTIVE` (Règle RG-07).
* **`Technicien` :** Représente l'opérateur technique qualifié en charge de l'exécution.
    * Attributs : `id`, `nom`, `specialite` (mécanique, électrique, hydraulique), `email`.
* **`Intervention` :** Matérialise l'ordre de maintenance liant un équipement et un technicien.
    * Attributs : `id`, `type`, `datePlanifiee`, `dateRealisation`, `dureeMin`, `cout`, `statut`, `description`.
    * Opération clé `terminer(date, duree, cout)` : Clôture formelle de l'intervention exigeant un coût positif, une durée non nulle et le statut initial `EN_COURS`.

### 2.2 Diagramme de Classes UML
Le diagramme est versionné sous format textuel PlantUML dans [`docs/uml/class-diagram.puml`](docs/uml/class-diagram.puml):

---

## 3. Règles de Gestion et Invariants OCL (Object Constraint Language)

Le système garantit l'intégrité de ses données à l'aide de contraintes d'intégrité formelles OCL (spécifiées dans [`docs/ocl.md`](docs/ocl.md)) :

1. **RG-01 (Unicité du code matériel) :** Deux équipements ne peuvent partager le même code fonctionnel.
2. **RG-02 (Coût positif) :** Tout coût saisi doit être supérieur ou égal à zéro ($cout \ge 0$).
3. **RG-03 (Cohérence temporelle) :** La `dateRealisation` ne peut jamais être antérieure à la `datePlanifiee`.
4. **RG-04 (Complétude de clôture) :** Une intervention `TERMINEE` exige impérativement une date de réalisation et une durée strictement positive ($dureeMin > 0$).
5. **RG-05 (Cohérence équipement en panne) :** Un équipement au statut `EN_PANNE` possède obligatoirement au moins une intervention non clôturée.
6. **RG-06 (Plafond de charge technicien) :** Un technicien ne peut avoir plus de 3 interventions simultanées au statut `EN_COURS`.
7. **RG-07 (Calcul du nombre de pannes) :** Le total des pannes subies est le décompte strict des interventions curatives rattachées.
8. **RG-08 (Verrouillage de clôture) :** Seule une intervention non terminée peut être clôturée via l'opération `terminer()`.

---

## 4. Architecture Technique Globale

L'application suit une architecture découplée **N-Tiers** garantissant séparation des responsabilités et testabilité :

---

## 5. Organisation du Dépôt

```text
maintenex/
├── backend/            # Code source du serveur d'application (Spring Boot, Java 21)
├── frontend/           # Code source du client web (Angular 22, TypeScript)
├── docs/               # Documentation fonctionnelle et technique
│   ├── uml/            # Fichiers sources PlantUML (.puml) et diagrammes générés
│   ├── sql/            # Scripts DDL, données de test (seed) et requêtes KPI Oracle
│   └── ocl.md          # Spécifications formelles des contraintes métier
└── README.md           # Documentation d'accueil et guide du projet
6. Feuille de Route et Méthodologie Agile
Le développement est séquencé selon le cadre Scrum en 5 itérations :   
PNG

Sprint 0 : Initialisation des environnements et outillage (Java 21, Node 24, Angular CLI, Git, Jira). (Validé)   
PNG

Sprint 1 : Modélisation conceptuelle (UML, OCL, User Stories) et schéma relationnel Oracle (Scripts DDL, seed data, requêtes d'agrégation). (En cours)   
PNG

Sprint 2 : Développement du backend Spring Boot (Couche domaine, services métier OCL, API REST, documentation OpenAPI / Swagger, tests unitaires JUnit 5).   
PDF
+ 1

Sprint 3 : Développement du frontend Angular (Interfaces de gestion, formulaires avec validation réactive, planning des 7 jours, tableau de bord visuel avec graphiques).   
PDF
+ 2

Sprint 4 : Intégration globale, matrice de traçabilité des exigences, validation des scénarios de test (T-01 à T-12) et préparation de la soutenance