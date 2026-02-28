# Service Clinical

## Vue d'ensemble

Le service **Clinical** est le coeur fonctionnel de Rofecare HIS. Il gere l'ensemble des activites cliniques : consultations, diagnostics, prescriptions, signes vitaux, et plans de traitement. C'est le service le plus volumineux du systeme avec 293 tests unitaires, refletant la complexite de la logique metier medicale.

| Propriete | Valeur |
|---|---|
| Port | `8083` |
| Base de donnees | `rofecare_clinical` (PostgreSQL, port `5434`) |
| Framework | Spring Boot 3.4.1, Java 21 |
| Architecture | Hexagonale / DDD |
| Tests unitaires | 293 |

## Architecture hexagonale

```
┌──────────────────────────────────────────────────────────────────┐
│                      infrastructure/                              │
│  ┌──────────────────────────┐  ┌───────────────────────────────┐ │
│  │  adapter/input/rest      │  │  adapter/output/persistence   │ │
│  │  - ConsultationController│  │  - ConsultationJpaRepository  │ │
│  │  - DiagnosisController   │  │  - DiagnosisJpaRepository     │ │
│  │  - PrescriptionController│  │  - PrescriptionJpaRepository  │ │
│  │  - VitalSignController   │  │  - VitalSignJpaRepository     │ │
│  │  - TreatmentPlanCtrl     │  │  - TreatmentPlanJpaRepository │ │
│  └──────────┬───────────────┘  └───────────────┬───────────────┘ │
│             │                                  │                 │
│  ┌──────────┴──────────────────────────────────┴───────────────┐ │
│  │  adapter/output/client                                      │ │
│  │  - PatientServiceClient (Feign)                             │ │
│  │  - IdentityServiceClient (Feign)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  config / security                                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                      application/                                 │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────────┐ │
│  │    port/      │ │     usecase/     │ │    dto / mapper       │ │
│  │  - InputPort  │ │ - ConsultationUC │ │  - ConsultationDTO    │ │
│  │  - OutputPort │ │ - DiagnosisUC    │ │  - DiagnosisDTO       │ │
│  │              │ │ - PrescriptionUC │ │  - PrescriptionDTO    │ │
│  │              │ │ - VitalSignUC    │ │  - VitalSignDTO       │ │
│  └──────────────┘ └──────────────────┘ └───────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  listener/                                                  │ │
│  │  - PatientEventListener                                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        domain/                                    │
│  ┌────────────────┐ ┌────────────┐ ┌──────────┐ ┌────────────┐  │
│  │     model       │ │ repository │ │  service  │ │   event    │  │
│  │ - Consultation  │ │            │ │           │ │            │  │
│  │ - Diagnosis     │ │            │ │           │ │            │  │
│  │ - Prescription  │ │            │ │           │ │            │  │
│  │ - VitalSign     │ │            │ │           │ │            │  │
│  │ - ClinicalNote  │ │            │ │           │ │            │  │
│  │ - TreatmentPlan │ │            │ │           │ │            │  │
│  │-MedicalProcedure│ │            │ │           │ │            │  │
│  │ - Observation   │ │            │ │           │ │            │  │
│  └────────────────┘ └────────────┘ └──────────┘ └────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Responsabilites

- **Gestion des consultations** : creation, mise a jour et cloture des consultations medicales
- **Gestion des diagnostics** : enregistrement des diagnostics utilisant les codes ICD-10
- **Gestion des prescriptions** : prescriptions de medicaments, ordonnances de laboratoire, demandes d'imagerie
- **Enregistrement des signes vitaux** : temperature, pression arterielle, frequence cardiaque, saturation en oxygene, etc.
- **Notes cliniques et observations** : documentation des observations medicales au cours du suivi
- **Plans de traitement** : elaboration et suivi des plans de traitement personnalises
- **Suivi des procedures medicales** : enregistrement et documentation des actes medicaux realises
- **Aide a la decision clinique** : alertes et suggestions basees sur les donnees cliniques du patient

## Modele de domaine

### Consultation
Represente une rencontre entre un praticien et un patient. Contient la date, le motif de consultation, le praticien responsable, le statut (en cours, completee, annulee), et les references vers les diagnostics et prescriptions associes.

### Diagnosis
Diagnostic medical associe a une consultation. Utilise le systeme de codification ICD-10 pour une standardisation internationale. Peut etre principal ou secondaire.

### Prescription
Ordonnance medicale emise lors d'une consultation. Peut concerner des medicaments (posologie, duree, frequence), des examens de laboratoire, ou des examens d'imagerie. Chaque prescription a un statut de suivi.

### VitalSign
Mesure des parametres vitaux du patient a un instant donne : temperature, tension arterielle (systolique/diastolique), frequence cardiaque, frequence respiratoire, saturation en oxygene (SpO2), poids, taille.

### ClinicalNote
Note textuelle libre saisie par le praticien pour documenter ses observations, reflexions cliniques, ou le deroulement de la consultation.

### TreatmentPlan
Plan de traitement structure pour un patient. Definit les objectifs therapeutiques, les interventions planifiees, les medicaments, et le calendrier de suivi.

### MedicalProcedure
Acte medical realise sur un patient (intervention chirurgicale mineure, ponction, suture, etc.). Inclut le type de procedure, le praticien, la date, et les notes associees.

### Observation
Observation clinique structuree. Contrairement aux notes cliniques libres, les observations suivent un format standardise permettant leur exploitation analytique.

## Endpoints API

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/clinical/consultations` | Lister les consultations (avec filtres) | Oui (`CONSULTATION_READ`) |
| `POST` | `/api/clinical/consultations` | Creer une nouvelle consultation | Oui (`CONSULTATION_CREATE`) |
| `GET` | `/api/clinical/consultations/{id}` | Obtenir les details d'une consultation | Oui (`CONSULTATION_READ`) |
| `PUT` | `/api/clinical/consultations/{id}` | Modifier une consultation | Oui (`CONSULTATION_UPDATE`) |
| `PUT` | `/api/clinical/consultations/{id}/complete` | Cloturer une consultation | Oui (`CONSULTATION_UPDATE`) |
| `GET` | `/api/clinical/consultations/{id}/diagnoses` | Lister les diagnostics d'une consultation | Oui (`DIAGNOSIS_READ`) |
| `POST` | `/api/clinical/diagnoses` | Enregistrer un diagnostic | Oui (`DIAGNOSIS_CREATE`) |
| `GET` | `/api/clinical/diagnoses/{id}` | Obtenir les details d'un diagnostic | Oui (`DIAGNOSIS_READ`) |
| `PUT` | `/api/clinical/diagnoses/{id}` | Modifier un diagnostic | Oui (`DIAGNOSIS_UPDATE`) |
| `POST` | `/api/clinical/prescriptions` | Creer une prescription | Oui (`PRESCRIPTION_CREATE`) |
| `GET` | `/api/clinical/prescriptions/{id}` | Obtenir les details d'une prescription | Oui (`PRESCRIPTION_READ`) |
| `PUT` | `/api/clinical/prescriptions/{id}` | Modifier une prescription | Oui (`PRESCRIPTION_UPDATE`) |
| `GET` | `/api/clinical/prescriptions/patient/{patientId}` | Lister les prescriptions d'un patient | Oui (`PRESCRIPTION_READ`) |
| `POST` | `/api/clinical/vital-signs` | Enregistrer des signes vitaux | Oui (`VITAL_SIGN_CREATE`) |
| `GET` | `/api/clinical/vital-signs/patient/{patientId}` | Historique des signes vitaux d'un patient | Oui (`VITAL_SIGN_READ`) |
| `GET` | `/api/clinical/vital-signs/latest/{patientId}` | Derniers signes vitaux d'un patient | Oui (`VITAL_SIGN_READ`) |
| `POST` | `/api/clinical/clinical-notes` | Ajouter une note clinique | Oui (`CLINICAL_NOTE_CREATE`) |
| `GET` | `/api/clinical/clinical-notes/consultation/{id}` | Notes cliniques d'une consultation | Oui (`CLINICAL_NOTE_READ`) |
| `POST` | `/api/clinical/treatment-plans` | Creer un plan de traitement | Oui (`TREATMENT_PLAN_CREATE`) |
| `GET` | `/api/clinical/treatment-plans/{id}` | Obtenir un plan de traitement | Oui (`TREATMENT_PLAN_READ`) |
| `PUT` | `/api/clinical/treatment-plans/{id}` | Modifier un plan de traitement | Oui (`TREATMENT_PLAN_UPDATE`) |
| `POST` | `/api/clinical/procedures` | Enregistrer une procedure medicale | Oui (`PROCEDURE_CREATE`) |
| `GET` | `/api/clinical/procedures/patient/{patientId}` | Procedures d'un patient | Oui (`PROCEDURE_READ`) |

## Evenements Kafka

### Evenements produits

| Topic | Evenement | Description | Payload |
|---|---|---|---|
| `clinical-events` | `consultation-completed` | Emis lorsqu'une consultation est cloturee | `consultationId`, `patientId`, `practitionerId`, `diagnoses`, `prescriptions`, `timestamp` |
| `clinical-events` | `prescription-ordered` | Emis lors de la creation d'une prescription | `prescriptionId`, `patientId`, `type` (medication/lab/imaging), `details`, `timestamp` |
| `clinical-events` | `diagnosis-recorded` | Emis lors de l'enregistrement d'un diagnostic | `diagnosisId`, `patientId`, `icdCode`, `description`, `timestamp` |
| `clinical-events` | `vital-signs-recorded` | Emis lors de l'enregistrement de signes vitaux | `patientId`, `vitalSigns`, `recordedBy`, `timestamp` |

### Evenements consommes

| Topic | Evenement | Description |
|---|---|---|
| `patient-events` | `patient-registered` | Recevoir les informations du patient pour les associer aux consultations |
| `patient-events` | `patient-admitted` | Mise a jour du contexte clinique lors de l'admission d'un patient |

## Feign Clients

| Service cible | Usage |
|---|---|
| **Patient** (`patient-service`) | Recuperation des informations du patient, verification du statut d'admission, acces au dossier medical |
| **Identity** (`identity-service`) | Verification de l'identite et des qualifications du praticien, validation des permissions |

## Configuration

```yaml
server:
  port: 8083

spring:
  datasource:
    url: jdbc:postgresql://localhost:5434/rofecare_clinical
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: clinical-service
      auto-offset-reset: earliest

feign:
  client:
    config:
      patient-service:
        url: http://localhost:8082
        connect-timeout: 5000
        read-timeout: 5000
      identity-service:
        url: http://localhost:8081
        connect-timeout: 5000
        read-timeout: 5000

jwt:
  secret: ${JWT_SECRET}
```

## Schema de base de donnees (tables principales)

| Table | Description |
|---|---|
| `consultations` | Consultations medicales |
| `diagnoses` | Diagnostics avec codes ICD-10 |
| `prescriptions` | Prescriptions (medicaments, laboratoire, imagerie) |
| `vital_signs` | Mesures des signes vitaux |
| `clinical_notes` | Notes cliniques textuelles |
| `treatment_plans` | Plans de traitement |
| `medical_procedures` | Procedures et actes medicaux |
| `observations` | Observations cliniques structurees |

## Tests

Le service Clinical dispose de **293 tests unitaires**, ce qui en fait le service le plus teste du systeme. Les tests couvrent :

- **JUnit 5** pour les tests unitaires
- **Mockito** pour le mocking des dependances
- **Testcontainers** pour les tests d'integration (PostgreSQL)
- Validation exhaustive de la logique de prescription
- Verification des regles de codification ICD-10
- Couverture des workflows de consultation (creation -> diagnostic -> prescription -> cloture)
- Tests des interactions avec les services Patient et Identity via des mocks Feign
