# Sous-domaines et Bounded Contexts

Ce document presente la decomposition strategique du systeme Rofecare selon les principes du **Domain-Driven Design (DDD)**.

---

## Table des matieres

- [Vue d'ensemble strategique](#vue-densemble-strategique)
- [Sous-domaines](#sous-domaines)
  - [Identity (Identite)](#identity-identite)
  - [Patient](#patient)
  - [Clinical (Clinique)](#clinical-clinique)
  - [Medical Technology (Technologie medicale)](#medical-technology-technologie-medicale)
  - [Pharmacy (Pharmacie)](#pharmacy-pharmacie)
  - [Finance](#finance)
  - [Platform (Plateforme)](#platform-plateforme)
  - [Interoperability (Interoperabilite)](#interoperability-interoperabilite)
- [Context Map](#context-map)
- [Flux d'evenements de domaine](#flux-devenements-de-domaine)
- [Glossaire du langage ubiquitaire](#glossaire-du-langage-ubiquitaire)

---

## Sous-domaines DNS (*.rofecare.com)

Chaque composant du systeme Rofecare est accessible via un sous-domaine dedie sur `rofecare.com`.

### Applications

| Sous-domaine | Service | Description |
|---|---|---|
| `rofecare.com` | Landing page | Site vitrine et page d'accueil |
| `app.rofecare.com` | Frontend Nuxt | Application web principale (SPA/SSR) |
| `api.rofecare.com` | API Gateway (8080) | Point d'entree unique pour toutes les API REST |
| `docs.rofecare.com` | Documentation | Documentation technique (GitHub Pages ou similaire) |

### Services metier (internes, via API Gateway)

Ces sous-domaines pointent vers le Gateway qui route vers le service correspondant.

| Sous-domaine | Route Gateway | Service | Port interne |
|---|---|---|---|
| `api.rofecare.com/identity` | `/api/identity/**` | Identity | 8081 |
| `api.rofecare.com/patients` | `/api/patients/**` | Patient | 8082 |
| `api.rofecare.com/clinical` | `/api/clinical/**` | Clinical | 8083 |
| `api.rofecare.com/medical-technology` | `/api/medical-technology/**` | Medical Technology | 8084 |
| `api.rofecare.com/pharmacy` | `/api/pharmacy/**` | Pharmacy | 8085 |
| `api.rofecare.com/finance` | `/api/finance/**` | Finance | 8086 |
| `api.rofecare.com/platform` | `/api/platform/**` | Platform | 8087 |
| `api.rofecare.com/interoperability` | `/api/interoperability/**` | Interoperability | 8088 |

### Infrastructure et monitoring

| Sous-domaine | Service | Description |
|---|---|---|
| `eureka.rofecare.com` | Discovery Server (8761) | Dashboard Eureka, registre des services |
| `config.rofecare.com` | Config Server (8888) | Configuration centralisee (acces restreint) |
| `grafana.rofecare.com` | Grafana (3001) | Dashboards de monitoring et metriques |
| `kibana.rofecare.com` | Kibana (5601) | Visualisation des logs centralises (ELK) |
| `zipkin.rofecare.com` | Zipkin (9411) | Tracing distribue des requetes |
| `kafka-ui.rofecare.com` | Kafka UI (8180) | Visualisation des topics Kafka |
| `prometheus.rofecare.com` | Prometheus (9090) | Collecte et stockage des metriques |

### Interoperabilite

| Sous-domaine | Service | Description |
|---|---|---|
| `fhir.rofecare.com` | Interoperability (8088) | Endpoint FHIR R4 pour les echanges standardises |
| `hl7.rofecare.com` | Interoperability (8088) | Endpoint HL7 v2 pour les messages de sante |

### Diagramme DNS

```
                          *.rofecare.com
                               |
        +----------+-----------+----------+-----------+
        |          |           |          |           |
   rofecare.com  app.       api.       docs.     grafana.
   (landing)   (frontend) (gateway)  (docs)    (monitoring)
                               |
              +------+------+------+------+------+------+------+
              |      |      |      |      |      |      |      |
          identity patient clinical medtech pharmacy finance platform interop
```

> **Note** : En production, seuls `rofecare.com`, `app.rofecare.com`, `api.rofecare.com`, `docs.rofecare.com` et `fhir.rofecare.com` sont exposes publiquement. Les sous-domaines d'infrastructure (`grafana`, `kibana`, `eureka`, etc.) sont accessibles uniquement via VPN ou reseau interne.

---

## Vue d'ensemble strategique

Le systeme Rofecare est decompose en **8 sous-domaines** qui correspondent chacun a un **Bounded Context** distinct, implemente par un microservice dedie.

### Classification des sous-domaines

| Sous-domaine        | Type             | Justification                                                |
|---------------------|------------------|--------------------------------------------------------------|
| Clinical            | **Core**         | Coeur du metier hospitalier : consultations et diagnostics   |
| Patient             | **Core**         | Gestion du parcours patient, element central du systeme      |
| Pharmacy            | **Core**         | Circuit du medicament, element critique de la chaine de soins|
| Medical Technology  | **Core**         | Laboratoire et imagerie, essentiels au diagnostic            |
| Finance             | **Supporting**   | Facturation et paiements, support au fonctionnement          |
| Identity            | **Supporting**   | Authentification et autorisation, transversal                |
| Platform            | **Generic**      | Notifications, audit, documents, fonctionnalites transversales|
| Interoperability    | **Generic**      | Standards externes (HL7 FHIR, DICOM), integration           |

---

## Sous-domaines

### Identity (Identite)

**Bounded Context : Gestion des identites et des acces**

Le sous-domaine Identity gere l'ensemble de la securite du systeme : authentification des utilisateurs, gestion des roles et permissions, et emission des tokens JWT.

#### Responsabilites

- Authentification des utilisateurs (login/logout)
- Gestion du cycle de vie des comptes utilisateurs
- Attribution et gestion des roles et permissions (RBAC)
- Emission et validation des tokens JWT
- Gestion des sessions et refresh tokens
- Politique de mots de passe et securite des comptes

#### Aggregats

| Aggregat       | Entites / Value Objects                                    |
|----------------|------------------------------------------------------------|
| **User**       | User (root), Credential, UserProfile                      |
| **Role**       | Role (root), Permission, RolePermission                    |
| **Token**      | RefreshToken (root), TokenClaims                           |

#### Invariants metier

- Un utilisateur doit avoir au moins un role actif
- Un token JWT expire apres une duree configurable
- Les mots de passe doivent respecter la politique de securite
- Un compte est verrouille apres N tentatives echouees

---

### Patient

**Bounded Context : Gestion du dossier patient**

Le sous-domaine Patient gere l'enregistrement, le triage, l'admission et le suivi des patients tout au long de leur parcours dans l'etablissement.

#### Responsabilites

- Enregistrement et identification des patients
- Gestion du dossier patient (informations demographiques, contacts)
- Triage aux urgences (classification de gravite)
- Gestion des admissions, transferts et sorties
- Historique des visites et sejours
- Gestion des contacts d'urgence et ayants droit

#### Aggregats

| Aggregat       | Entites / Value Objects                                    |
|----------------|------------------------------------------------------------|
| **Patient**    | Patient (root), PatientIdentifier, Address, ContactInfo, EmergencyContact |
| **Visit**      | Visit (root), Admission, Transfer, Discharge               |
| **Triage**     | Triage (root), TriageAssessment, VitalSigns, PriorityLevel |

#### Invariants metier

- Chaque patient possede un identifiant unique dans le systeme
- Un patient ne peut avoir qu'une seule admission active a la fois
- Le triage attribue obligatoirement un niveau de priorite
- Les informations d'identite sont obligatoires (nom, date de naissance, sexe)

---

### Clinical (Clinique)

**Bounded Context : Activite clinique et soins**

Sous-domaine central du systeme, Clinical couvre l'ensemble de l'activite medicale : consultations, examens, diagnostics, prescriptions et plans de traitement.

#### Responsabilites

- Creation et gestion des consultations medicales
- Enregistrement des examens cliniques et observations
- Pose de diagnostics (codes CIM-10/CIM-11)
- Redaction des prescriptions medicamenteuses
- Definition des plans de traitement
- Suivi des constantes vitales
- Gestion des antecedents medicaux

#### Aggregats

| Aggregat          | Entites / Value Objects                                    |
|-------------------|------------------------------------------------------------|
| **Consultation**  | Consultation (root), ClinicalExam, Observation, VitalSigns |
| **Diagnosis**     | Diagnosis (root), DiagnosisCode (ICD-10), DiagnosisType    |
| **Prescription**  | Prescription (root), PrescriptionItem, Dosage, Duration    |
| **TreatmentPlan** | TreatmentPlan (root), TreatmentStep, Goal                  |
| **MedicalHistory**| MedicalHistory (root), Allergy, ChronicCondition, Surgery  |

#### Invariants metier

- Toute consultation doit etre liee a un patient et un medecin
- Une prescription ne peut etre emise que dans le cadre d'une consultation
- Les codes diagnostics doivent etre valides (CIM-10/CIM-11)
- Un medecin ne peut modifier que ses propres consultations (sauf superviseur)

---

### Medical Technology (Technologie medicale)

**Bounded Context : Laboratoire, imagerie et equipements**

Ce sous-domaine gere les activites de laboratoire (analyses biologiques), d'imagerie medicale et la gestion du parc d'equipements medicaux.

#### Responsabilites

- Gestion des demandes d'examens de laboratoire
- Saisie et validation des resultats d'analyses
- Gestion des demandes d'imagerie medicale
- Stockage et consultation des images (integration DICOM)
- Gestion du parc d'equipements medicaux
- Controle qualite des analyses
- Gestion des reactifs et consommables de laboratoire

#### Aggregats

| Aggregat         | Entites / Value Objects                                    |
|------------------|------------------------------------------------------------|
| **LabOrder**     | LabOrder (root), LabTest, Sample, SampleType               |
| **LabResult**    | LabResult (root), TestResult, ReferenceRange, Interpretation|
| **ImagingOrder** | ImagingOrder (root), ImagingStudy, Modality                |
| **Equipment**    | Equipment (root), MaintenanceRecord, CalibrationLog        |

#### Invariants metier

- Un resultat de laboratoire doit etre valide par un biologiste
- Les valeurs hors normes declenchent un signalement automatique
- Un equipement en maintenance ne peut pas etre utilise pour des examens
- Les echantillons doivent etre traces de bout en bout

---

### Pharmacy (Pharmacie)

**Bounded Context : Circuit du medicament**

Le sous-domaine Pharmacy gere l'integralite du circuit du medicament : catalogue, stock, dispensation, et suivi des traitements.

#### Responsabilites

- Gestion du catalogue de medicaments (DCI, specialites)
- Gestion des stocks (entrees, sorties, inventaires)
- Reception et validation des prescriptions
- Dispensation des medicaments aux patients
- Suivi des dates de peremption
- Gestion des commandes fournisseurs
- Alertes de stock minimum et ruptures

#### Aggregats

| Aggregat         | Entites / Value Objects                                    |
|------------------|------------------------------------------------------------|
| **Medication**   | Medication (root), ActiveIngredient, DosageForm, RouteOfAdmin|
| **Stock**        | StockItem (root), StockMovement, BatchNumber, ExpiryDate   |
| **Dispensing**   | Dispensing (root), DispensedItem, SubstitutionReason        |
| **PurchaseOrder**| PurchaseOrder (root), OrderLine, Supplier                   |

#### Invariants metier

- La dispensation requiert une prescription valide et non expiree
- Le stock ne peut pas etre negatif
- Les medicaments perimes ne peuvent pas etre dispenses
- Toute substitution de medicament doit etre justifiee et tracee

---

### Finance

**Bounded Context : Facturation et gestion financiere**

Le sous-domaine Finance gere la facturation des actes medicaux, les paiements, la gestion des couvertures d'assurance et les rapports financiers.

#### Responsabilites

- Generation des factures a partir des actes realises
- Gestion des tarifs et nomenclature des actes
- Suivi des paiements (especes, mobile money, carte)
- Gestion des prises en charge par les assurances (tiers payant)
- Generation des bordereaux de facturation assurance
- Rapports financiers et tableaux de bord
- Gestion de la caisse et cloture journaliere

#### Aggregats

| Aggregat          | Entites / Value Objects                                    |
|-------------------|------------------------------------------------------------|
| **Invoice**       | Invoice (root), InvoiceLine, TariffCode, Amount            |
| **Payment**       | Payment (root), PaymentMethod, Receipt                     |
| **InsuranceClaim**| InsuranceClaim (root), ClaimItem, CoverageRate             |
| **Tariff**        | Tariff (root), TariffCategory, PriceSchedule               |
| **CashRegister**  | CashRegister (root), CashSession, DailyClosing             |

#### Invariants metier

- Une facture doit referencer au moins un acte medical
- Un paiement ne peut pas depasser le montant restant du
- Le taux de couverture assurance ne peut pas depasser 100%
- La cloture de caisse verifie la concordance des montants

---

### Platform (Plateforme)

**Bounded Context : Services transversaux**

Le sous-domaine Platform fournit les services transversaux necessaires au bon fonctionnement du systeme : notifications, audit, gestion documentaire et planification.

#### Responsabilites

- Envoi de notifications (email, SMS, push, in-app)
- Journalisation et audit des actions utilisateurs
- Gestion documentaire (rapports, attestations, certificats)
- Planification des rendez-vous et des ressources
- Generation de rapports et statistiques
- Gestion des templates de documents

#### Aggregats

| Aggregat          | Entites / Value Objects                                    |
|-------------------|------------------------------------------------------------|
| **Notification**  | Notification (root), NotificationChannel, Template         |
| **AuditLog**      | AuditEntry (root), ActionType, ActorRef, EntityRef         |
| **Document**      | Document (root), DocumentType, DocumentContent, Signature  |
| **Appointment**   | Appointment (root), TimeSlot, Resource, RecurrenceRule     |

#### Invariants metier

- Les actions critiques (prescriptions, dispensations) doivent etre auditees
- Un rendez-vous ne peut pas chevaucher un autre sur la meme ressource
- Les notifications echouees sont reessayees avec backoff exponentiel
- Les documents signes ne peuvent plus etre modifies

---

### Interoperability (Interoperabilite)

**Bounded Context : Integration avec les systemes externes**

Le sous-domaine Interoperability gere les echanges avec les systemes externes via les standards de sante (HL7 FHIR, DICOM) et les mecanismes d'integration (webhooks).

#### Responsabilites

- Exposition des ressources FHIR (Patient, Encounter, Observation, etc.)
- Reception et traitement des messages HL7
- Integration DICOM pour l'imagerie medicale
- Gestion des webhooks entrants et sortants
- Mapping entre le modele interne et les standards externes
- Gestion des partenaires et de leurs configurations d'integration

#### Aggregats

| Aggregat          | Entites / Value Objects                                    |
|-------------------|------------------------------------------------------------|
| **FhirResource**  | FhirResource (root), ResourceType, FhirVersion            |
| **Integration**   | Integration (root), Partner, Endpoint, AuthConfig          |
| **Webhook**       | Webhook (root), WebhookEvent, DeliveryAttempt              |
| **DicomStudy**    | DicomStudy (root), DicomSeries, DicomInstance, Modality     |

#### Invariants metier

- Les ressources FHIR doivent etre conformes au profil declare
- Les webhooks echoues sont reessayes selon une politique configurable
- Les mappings de donnees doivent etre bidirectionnels et sans perte
- Les connexions externes utilisent obligatoirement TLS

---

## Context Map

Le diagramme suivant represente les relations entre les Bounded Contexts du systeme Rofecare.

```
+-------------------------------------------------------------------+
|                        CONTEXT MAP                                 |
+-------------------------------------------------------------------+

                    +-------------------+
                    |     Identity      |
                    | (Authentication)  |
                    +--------+----------+
                             |
                    [OHS] Open Host Service
                    (JWT validation pour tous les services)
                             |
         +-------------------+-------------------+
         |                   |                   |
         v                   v                   v
+--------+------+   +--------+------+   +--------+------+
|    Patient    |   |   Clinical    |   |   Platform    |
| (Dossier pat.)|   | (Soins med.)  |   | (Transversal) |
+-------+-------+   +---+------+---+   +--------+------+
        |                |      |                |
        |    [ACL]       |      |     [ACL]      |
        +<-- Customer ---+      +--- Customer -->+
        |   /Supplier    |      |   /Supplier    |
        |                |      |                |
        |           +----+----+ |                |
        |           |         | |                |
        |    +------+------+  | |    +-----------+----------+
        |    |  Pharmacy   |  | |    |   Interoperability   |
        |    | (Medicament)|  | |    | (Standards externes) |
        |    +------+------+  | |    +-----------+----------+
        |           |         | |                |
        |    [CF]   |         | |         [ACL]  |
        |  Conformist         | |    Published Language
        |           |         | |    (FHIR, DICOM, HL7)
        |           v         v v                |
        |    +------+---------+------+           |
        +---->      Finance          <-----------+
             | (Facturation)         |
             +-----------+-----------+
                         |
                  +------+------+
                  |  MedTech    |
                  | (Labo/Img) |
                  +-------------+
```

### Relations entre contextes

| Relation                   | Type                  | Description                                           |
|----------------------------|-----------------------|-------------------------------------------------------|
| Identity -> tous           | **Open Host Service** | Fournit l'authentification JWT a tous les services     |
| Clinical -> Patient        | **Customer/Supplier** | Clinical consomme les donnees patient                  |
| Clinical -> Pharmacy       | **Customer/Supplier** | Clinical emet les prescriptions, Pharmacy les traite   |
| Clinical -> MedTech        | **Customer/Supplier** | Clinical demande des examens, MedTech fournit les resultats |
| Finance <- Clinical        | **Conformist**        | Finance se conforme aux actes definis par Clinical     |
| Finance <- Pharmacy        | **Conformist**        | Finance se conforme aux dispensations de Pharmacy      |
| Finance <- MedTech         | **Conformist**        | Finance se conforme aux examens realises par MedTech   |
| Platform <- tous           | **Customer/Supplier** | Platform recoit les evenements de tous les services    |
| Interop <-> externe        | **Published Language**| Utilise les standards FHIR, DICOM, HL7                 |
| Interop -> Patient/Clinical| **ACL**               | Couche anti-corruption pour mapper les standards       |

### Types de relations utilisees

- **OHS (Open Host Service)** : le fournisseur expose un service ouvert et standardise
- **Customer/Supplier** : relation contractuelle, le fournisseur s'adapte au client
- **Conformist** : le consommateur se conforme au modele du fournisseur
- **ACL (Anti-Corruption Layer)** : couche de traduction pour proteger le modele interne
- **Published Language** : utilisation de standards industriels comme langage commun

---

## Flux d'evenements de domaine

Les evenements de domaine permettent la communication asynchrone entre les Bounded Contexts. Voici les flux principaux.

### Parcours patient complet

```
1. Enregistrement
   Patient ──[patient-registered]──> Clinical    (creer dossier medical)
                                 ──> Finance     (creer compte financier)
                                 ──> Platform    (notification, audit)

2. Consultation
   Clinical ──[consultation-completed]──> Finance   (facturer la consultation)
                                      ──> Platform  (notification, audit)

3. Prescription
   Clinical ──[prescription-ordered]──> Pharmacy   (preparer les medicaments)

4. Dispensation
   Pharmacy ──[medication-dispensed]──> Finance    (facturer les medicaments)
                                    ──> Clinical   (mettre a jour le dossier)

5. Examen de laboratoire
   MedTech ──[laboratory-result-available]──> Clinical  (integrer les resultats)
                                           ──> Platform (notification au medecin)

6. Facturation
   Finance ──[invoice-created]──> Platform   (envoyer la facture au patient)

7. Creation utilisateur
   Identity ──[user-created]──> Platform   (notification de bienvenue, audit)
```

### Diagramme de sequence

```
Patient    Clinical    Pharmacy    MedTech    Finance    Platform
  |            |           |          |          |          |
  |--register->|           |          |          |          |
  |  [patient-registered]  |          |          |          |
  |            |-----------|----------|--------->|          |
  |            |-----------|----------|----------|--------->|
  |            |           |          |          |          |
  |--consult-->|           |          |          |          |
  |  [consultation-completed]         |          |          |
  |            |-----------|----------|--------->|          |
  |            |-----------|----------|----------|--------->|
  |            |           |          |          |          |
  |            |--prescribe->         |          |          |
  |            | [prescription-ordered]          |          |
  |            |           |          |          |          |
  |            |    <--dispense--     |          |          |
  |            |  [medication-dispensed]         |          |
  |            |           |----------|--------->|          |
  |            |           |          |          |          |
  |            |     <--lab-result----|          |          |
  |            |  [lab-result-available]         |          |
  |            |           |          |----------|--------->|
  |            |           |          |          |          |
```

---

## Glossaire du langage ubiquitaire

Chaque Bounded Context possede son propre vocabulaire. Les termes suivants constituent le langage ubiquitaire de Rofecare.

### Identity

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **User**          | Personne ayant un compte dans le systeme                            |
| **Role**          | Ensemble nomme de permissions attribuees a un utilisateur           |
| **Permission**    | Droit d'effectuer une action specifique dans le systeme             |
| **Credential**    | Informations d'authentification (mot de passe, token)               |
| **JWT**           | JSON Web Token utilise pour l'authentification sans etat            |

### Patient

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Patient**       | Personne recevant des soins dans l'etablissement                    |
| **Visit**         | Passage d'un patient dans l'etablissement (ambulatoire ou hospitalisation) |
| **Admission**     | Accueil formel d'un patient pour une hospitalisation                |
| **Triage**        | Evaluation initiale de la gravite aux urgences                      |
| **Discharge**     | Sortie officielle d'un patient apres traitement                     |
| **Transfer**      | Deplacement d'un patient d'un service a un autre                    |

### Clinical

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Consultation**  | Rencontre formelle entre un medecin et un patient                   |
| **Diagnosis**     | Identification d'une maladie ou condition (code CIM-10)             |
| **Prescription**  | Ordonnance medicale prescrivant des medicaments                     |
| **Vital Signs**   | Constantes vitales (pouls, tension, temperature, saturation)        |
| **Clinical Exam** | Examen physique realise par le medecin                              |
| **Treatment Plan**| Plan structure de soins pour un patient                             |

### Medical Technology

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Lab Order**     | Demande d'examens de laboratoire                                    |
| **Lab Result**    | Resultat d'une analyse de laboratoire                               |
| **Sample**        | Echantillon biologique preleve sur un patient                       |
| **Imaging Order** | Demande d'examen d'imagerie medicale                                |
| **Modality**      | Type d'equipement d'imagerie (radiographie, echographie, IRM, etc.) |
| **Reference Range** | Plage de valeurs normales pour un parametre biologique            |

### Pharmacy

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Medication**    | Medicament enregistre dans le catalogue de la pharmacie             |
| **Dispensing**    | Acte de delivrer un medicament a un patient                         |
| **Stock Movement**| Entree ou sortie de stock de medicaments                            |
| **Batch Number**  | Numero de lot d'un medicament pour la tracabilite                   |
| **DCI**           | Denomination Commune Internationale (nom generique du principe actif)|
| **Substitution**  | Remplacement d'un medicament par un equivalent                      |

### Finance

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Invoice**       | Facture emise pour des actes medicaux realises                      |
| **Tariff**        | Grille tarifaire des actes et prestations                           |
| **Payment**       | Reglement d'une facture (partiel ou total)                          |
| **Insurance Claim**| Demande de remboursement aupres d'un organisme d'assurance         |
| **Coverage Rate** | Pourcentage de prise en charge par l'assurance                      |
| **Cash Register** | Caisse gerant les encaissements quotidiens                          |

### Platform

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **Notification**  | Message envoye a un utilisateur ou patient via un canal             |
| **Audit Entry**   | Trace d'une action effectuee dans le systeme                        |
| **Document**      | Fichier genere par le systeme (rapport, attestation, certificat)    |
| **Appointment**   | Rendez-vous planifie pour un patient ou une ressource               |
| **Template**      | Modele de document ou de notification reutilisable                  |

### Interoperability

| Terme             | Definition                                                          |
|-------------------|---------------------------------------------------------------------|
| **FHIR Resource** | Ressource au format HL7 FHIR (Patient, Encounter, Observation)     |
| **HL7**           | Health Level Seven, standard d'echange de donnees de sante          |
| **DICOM**         | Digital Imaging and Communications in Medicine                       |
| **Webhook**       | Mecanisme de notification HTTP pour les integrations                |
| **Integration**   | Configuration de connexion avec un systeme externe                  |
| **Partner**       | Systeme ou etablissement externe connecte                           |

---

*Voir aussi :*
- [Vue d'ensemble de l'architecture](overview.md)
- [Strategie de deploiement dual](dual-deployment.md)
