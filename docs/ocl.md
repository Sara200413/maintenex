# Spécification Formelle des Règles Métier en OCL (Maintenex / GearFlow)

Ce document formalise les 8 règles de gestion critiques du système de GMAO à l'aide du langage **OCL (Object Constraint Language)**. Ces contraintes d'intégrité définissent les règles appliquées dans la base de données Oracle 21c (contraintes CHECK et déclencheurs), la couche de domaine Java/Spring Boot (services et annotations de validation), ainsi que les formulaires de l'interface Angular.

---

## Synthèse des Contraintes et Règles de Gestion

| Réf. | Règle Métier | Invariant OCL | Cible UML | Traduction Oracle |
| :--- | :--- | :--- | :--- | :--- |
| **RG-01** | Unicité du code d'équipement | `codeUnique` | `Equipement` | `UNIQUE(code)` |
| **RG-02** | Coût d'intervention non négatif | `coutPositif` | `Intervention` | `CHECK (cout >= 0)` |
| **RG-03** | Chronologie des dates | `dateCoherente` | `Intervention` | `CHECK (date_realisation >= date_planifiee)` |
| **RG-04** | Complétude d'une intervention terminée | `termineeComplete` | `Intervention` | `CHECK (date et duree obligatoires)` |
| **RG-05** | Couverture de panne obligatoire | `enPanneOuverte` | `Equipement` | Logique applicative Service |
| **RG-06** | Plafond de 3 interventions en cours | `chargeMax` | `Technicien` | Service (HTTP 409) |
| **RG-07** | Calcul dérivé du nombre de pannes | `nbPannes` | `Equipement` | Agrégation SQL `COUNT(*)` |
| **RG-08** | Verrouillage de clôture | `terminer()` | `Intervention` | Précondition Service (HTTP 409) |

---

## 1. Règle RG-01 : Unicité du Code Équipement

* **Description métier :** Deux équipements du parc industriel ne peuvent jamais porter le même code d'identification technique.
* **Spécification OCL :**
```ocl
context Equipement inv codeUnique:
    Equipement.allInstances()->isUnique(code)
```
* **Mise en œuvre technique :**
    * **Oracle :** Contrainte d'intégrité `CONSTRAINT uq_equipement_code UNIQUE (code)`.
    * **Spring Boot :** Vérification dans `EquipementService.creer()` et renvoi d'une exception `DuplicateResourceException` (`HTTP 409 Conflict`).
    * **Angular :** Contrôle asynchrone lors de la saisie dans le formulaire d'ajout.

---

## 2. Règle RG-02 : Positivité du Coût d'Intervention

* **Description métier :** Le coût financier engagé pour une intervention doit être supérieur ou égal à zéro ($cout \ge 0$).
* **Spécification OCL :**
```ocl
context Intervention inv coutPositif:
    self.cout >= 0
```
* **Mise en œuvre technique :**
    * **Oracle :** Contrainte de table `CONSTRAINT chk_intervention_cout CHECK (cout >= 0)`.
    * **Spring Boot :** Annotation Bean Validation `@PositiveOrZero(message = "Le coût doit être positif ou nul")`.
    * **Angular :** Validateur de contrôle `Validators.min(0)`.

---

## 3. Règle RG-03 : Cohérence Chronologique des Dates

* **Description métier :** Lorsqu'une date de réalisation effective est renseignée, elle ne peut pas être antérieure à la date de planification initiale.
* **Spécification OCL :**
```ocl
context Intervention inv dateCoherente:
    self.dateRealisation->notEmpty() implies self.dateRealisation >= self.datePlanifiee
```
* **Mise en œuvre technique :**
    * **Oracle :** Contrainte `CONSTRAINT chk_intervention_dates CHECK (date_realisation IS NULL OR date_realisation >= date_planifiee)`.
    * **Spring Boot :** Vérification croisée dans le service lors du traitement de la clôture.
    * **Angular :** Validateur personnalisé au niveau du `FormGroup`.

---

## 4. Règle RG-04 : Complétude d'une Intervention Clôturée

* **Description métier :** Une intervention ayant le statut `TERMINEE` exige obligatoirement la saisie d'une date de réalisation effective et d'une durée strictement positive ($dureeMin > 0$).
* **Spécification OCL :**
```ocl
context Intervention inv termineeComplete:
    self.statut = StatutIntervention::TERMINEE implies (self.dateRealisation->notEmpty() and self.dureeMin > 0)
```
* **Mise en œuvre technique :**
    * **Oracle :** Contrainte `CONSTRAINT chk_intervention_cloture CHECK (statut != 'TERMINEE' OR (date_realisation IS NOT NULL AND duree_min > 0))`.
    * **Spring Boot :** Contrôle dans la méthode métier `Intervention.terminer(date, duree, cout)`.
    * **Angular :** Champs requis avec indicateur d'erreur dans le modal de clôture.

---

## 5. Règle RG-05 : Prise en Charge d'un Équipement en Panne

* **Description métier :** Tout équipement dont le statut est `EN_PANNE` doit être couvert par au moins une intervention non encore terminée (statut différent de `TERMINEE`).
* **Spécification OCL :**
```ocl
context Equipement inv enPanneOuverte:
    self.statut = StatutEquipement::EN_PANNE implies self.interventions->exists(i | i.statut <> StatutIntervention::TERMINEE)
```
* **Mise en œuvre technique :**
    * **Spring Boot :** Lors du basculement d'un équipement en statut `EN_PANNE`, le service déclenche automatiquement la création d'une intervention corrective ou valide l'existence d'une intervention ouverte.

---

## 6. Règle RG-06 : Seuil Maximal de Charge par Technicien

* **Description métier :** Un technicien de maintenance ne peut avoir plus de 3 interventions simultanées au statut `EN_COURS`.
* **Spécification OCL :**
```ocl
context Technicien inv chargeMax:
    self.interventions->select(i | i.statut = StatutIntervention::EN_COURS)->size() <= 3
```
* **Mise en œuvre technique :**
    * **Spring Boot :** Lors de l'appel à `demarrer(interventionId)`, le service calcule le volume de tâches en cours du technicien et rejette l'opération avec un code `HTTP 409 Conflict` si le seuil de 3 est déjà atteint.
    * **Angular :** Désactivation de l'action de démarrage et affichage d'une pastille d'alerte lorsque le technicien est surchargé.

---

## 7. Règle RG-07 : Calcul Dérivé du Nombre de Pannes

* **Description métier :** L'attribut dérivé `nbPannes` d'un équipement est égal au comptage exact de ses interventions rattachées dont la typologie est `CORRECTIVE`.
* **Spécification OCL :**
```ocl
context Equipement::nbPannes : Integer
derive:
    self.interventions->select(i | i.type = TypeIntervention::CORRECTIVE)->size()
```
* **Mise en œuvre technique :**
    * **Oracle :** Requête SQL analytique :
      ```sql
      SELECT equipement_id, COUNT(*) AS nb_pannes
      FROM intervention
      WHERE type = 'CORRECTIVE'
      GROUP BY equipement_id;
      ```
    * **Spring Boot :** Calcul dynamique ou projection JPA exposée dans `EquipementResponseDTO`.

---

## 8. Règle RG-08 : Précondition et Postcondition de Clôture

* **Description métier :** La méthode `terminer(...)` ne peut s'exécuter que sur une intervention non encore clôturée, et garantit que le statut final passe à `TERMINEE`.
* **Spécification OCL :**
```ocl
context Intervention::terminer(d : LocalDate, duree : Integer, cout : Real)
pre interventionNonTerminee:
    self.statut <> StatutIntervention::TERMINEE
post interventionCloturee:
    self.statut = StatutIntervention::TERMINEE
```
* **Mise en œuvre technique :**
    * **Spring Boot :** Test initial dans `InterventionService.terminer(...)` avec levée d'une `IllegalStateException` (`HTTP 409 Conflict`) si l'intervention est déjà clôturée.
    * **Angular :** Désactivation visuelle immédiate du bouton d'action dès que le statut passe à `TERMINEE`.