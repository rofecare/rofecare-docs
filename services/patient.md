# Service Patient

## Vue d'ensemble

Le service **Patient** gere l'ensemble du cycle de vie du patient au sein de l'etablissement hospitalier : de l'enregistrement initial jusqu'a la sortie, en passant par les admissions, transferts et le triage. Il constitue le referentiel central des donnees demographiques et du dossier medical de chaque patient.

| Propriete | Valeur |
|---|---|
| Port | `8082` |
| Base de donnees | `rofecare_patient` (PostgreSQL, port `5433`) |
| Framework | Spring Boot 3.4.1, Java 21 |
| Architecture | Hexagonale / DDD |

## Architecture hexagonale

```
┌──────────────────────────────────────────────────────────────┐
│                     infrastructure/                           │
│  ┌────────────────────────┐  ┌─────────────────────────────┐ │
│  │  adapter/input/rest    │  │ adapter/output/persistence   │ │
│  │  - PatientController   │  │ - PatientJpaRepository       │ │
│  │  - AdmissionController │  │ - AdmissionJpaRepository     │ │
│  │  - TriageController    │  │ - MedicalRecordJpaRepository │ │
│  └──────────┬─────────────┘  └──────────────┬──────────────┘ │
│             │                               │                │
│  ┌──────────┴───────────────────────────────┴──────────────┐ │
│  │  adapter/output/client                                  │ │
│  │  - IdentityServiceClient (Feign)                        │ │
│  └─────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  config / security                                      │ │
│  └─────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────┤
│                     application/                              │
│  ┌──────────────┐ ┌────────────────┐ ┌────────────────────┐  │
│  │    port/      │ │    usecase/    │ │   dto / mapper     │  │
│  │  - InputPort  │ │ - RegisterUC   │ │  - PatientDTO      │  │
│  │  - OutputPort │ │ - AdmitUC      │ │  - AdmissionDTO    │  │
│  │              │ │ - TriageUC     │ │  - TriageDTO       │  │
│  └──────────────┘ └────────────────┘ └────────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│                       domain/                                 │
│  ┌──────────────┐ ┌────────────┐ ┌──────────┐ ┌──────────┐  │
│  │    model      │ │ repository │ │  service  │ │  event   │  │
│  │  - Patient    │ │            │ │           │ │          │  │
│  │  - Admission  │ │            │ │           │ │          │  │
│  │  - Discharge  │ │            │ │           │ │          │  │
│  │  - Transfer   │ │            │ │           │ │          │  │
│  │  - Triage     │ │            │ │           │ │          │  │
│  └──────────────┘ └────────────┘ └──────────┘ └──────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## Responsabilites

- **Gestion des donnees demographiques** : enregistrement et mise a jour des informations personnelles du patient (nom, date de naissance, adresse, contact)
- **Gestion du dossier medical** : creation et consultation du dossier medical complet
- **Enregistrement des patients** : enregistrement normal et enregistrement d'urgence (processus simplifie)
- **ADT (Admission, Discharge, Transfer)** : gestion complete du parcours hospitalier du patient
- **Triage** : classification d'urgence des patients selon la gravite
- **Recherche de patients** : recherche multi-criteres (nom, identifiant, numero d'assurance)
- **Gestion des informations d'assurance** : stockage et validation des couvertures d'assurance
- **Contacts d'urgence** : gestion des personnes a contacter et proches du patient

## Modele de domaine

### Patient
Entite principale representant un patient. Contient les donnees demographiques, le statut (enregistre, admis, sorti), et les liens vers le dossier medical, les informations d'assurance et les contacts d'urgence.

### MedicalRecord
Dossier medical du patient. Regroupe l'historique medical, les allergies connues, les antecedents, et les references vers les consultations et traitements.

### Admission
Represente l'admission d'un patient dans un service hospitalier. Contient la date d'admission, le service, le lit assigne, le motif d'admission et le medecin responsable.

### Discharge
Enregistre la sortie d'un patient. Inclut la date de sortie, le motif, les instructions de suivi et le resume de sortie.

### Transfer
Represente le transfert d'un patient d'un service a un autre au sein de l'etablissement. Contient le service d'origine, le service de destination, et le motif du transfert.

### Triage
Classification d'urgence d'un patient a son arrivee. Utilise un systeme de niveaux de priorite pour determiner l'ordre de prise en charge.

### InsuranceInfo
Informations relatives a la couverture d'assurance du patient : organisme, numero de police, dates de validite, type de couverture.

### EmergencyContact
Contact a prevenir en cas d'urgence. Contient le nom, le lien de parente, et les coordonnees.

### NextOfKin
Proche du patient designe comme personne de reference. Similaire au contact d'urgence mais avec un role administratif et legal elargi.

## Endpoints API

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/patients` | Lister les patients (avec pagination et filtres) | Oui (`PATIENT_READ`) |
| `POST` | `/api/patients` | Enregistrer un nouveau patient | Oui (`PATIENT_CREATE`) |
| `GET` | `/api/patients/{id}` | Obtenir les details d'un patient | Oui (`PATIENT_READ`) |
| `PUT` | `/api/patients/{id}` | Modifier les informations d'un patient | Oui (`PATIENT_UPDATE`) |
| `GET` | `/api/patients/search` | Rechercher des patients (nom, ID, assurance) | Oui (`PATIENT_READ`) |
| `POST` | `/api/patients/{id}/admissions` | Admettre un patient | Oui (`ADMISSION_CREATE`) |
| `GET` | `/api/patients/{id}/admissions` | Lister les admissions d'un patient | Oui (`ADMISSION_READ`) |
| `PUT` | `/api/patients/{id}/admissions/{admissionId}` | Modifier une admission | Oui (`ADMISSION_UPDATE`) |
| `POST` | `/api/patients/{id}/discharge` | Enregistrer la sortie d'un patient | Oui (`DISCHARGE_CREATE`) |
| `POST` | `/api/patients/{id}/transfer` | Transferer un patient | Oui (`TRANSFER_CREATE`) |
| `GET` | `/api/patients/{id}/medical-records` | Consulter le dossier medical d'un patient | Oui (`MEDICAL_RECORD_READ`) |
| `POST` | `/api/patients/{id}/medical-records` | Ajouter au dossier medical | Oui (`MEDICAL_RECORD_CREATE`) |
| `POST` | `/api/patients/triage` | Effectuer un triage de patient | Oui (`TRIAGE_CREATE`) |
| `GET` | `/api/patients/triage` | Lister les patients en attente de triage | Oui (`TRIAGE_READ`) |
| `PUT` | `/api/patients/{id}/insurance` | Mettre a jour les informations d'assurance | Oui (`PATIENT_UPDATE`) |
| `GET` | `/api/patients/{id}/contacts` | Lister les contacts d'urgence d'un patient | Oui (`PATIENT_READ`) |
| `POST` | `/api/patients/{id}/contacts` | Ajouter un contact d'urgence | Oui (`PATIENT_UPDATE`) |

## Evenements Kafka

### Evenements produits

| Topic | Evenement | Description | Payload |
|---|---|---|---|
| `patient-events` | `patient-registered` | Emis lors de l'enregistrement d'un nouveau patient | `patientId`, `name`, `dateOfBirth`, `registrationType`, `timestamp` |
| `patient-events` | `patient-admitted` | Emis lors de l'admission d'un patient | `patientId`, `admissionId`, `department`, `admittingDoctor`, `timestamp` |
| `patient-events` | `patient-discharged` | Emis lors de la sortie d'un patient | `patientId`, `dischargeId`, `reason`, `timestamp` |
| `patient-events` | `patient-transferred` | Emis lors du transfert d'un patient | `patientId`, `transferId`, `fromDepartment`, `toDepartment`, `timestamp` |

### Evenements consommes

| Topic | Evenement | Description |
|---|---|---|
| `user-events` | `user-created` | Reception pour associer les praticiens aux dossiers patients |

## Feign Clients

| Service cible | Usage |
|---|---|
| **Identity** (`identity-service`) | Verification des permissions utilisateur, validation des tokens, resolution des identites des praticiens |

## Configuration

```yaml
server:
  port: 8082

spring:
  datasource:
    url: jdbc:postgresql://localhost:5433/rofecare_patient
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
      group-id: patient-service
      auto-offset-reset: earliest

feign:
  client:
    config:
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
| `patients` | Donnees demographiques des patients |
| `medical_records` | Dossiers medicaux |
| `admissions` | Enregistrements des admissions |
| `discharges` | Enregistrements des sorties |
| `transfers` | Historique des transferts |
| `triage_records` | Classifications de triage |
| `insurance_info` | Informations d'assurance |
| `emergency_contacts` | Contacts d'urgence |
| `next_of_kin` | Proches designes |

## Tests

Le service utilise :

- **JUnit 5** pour les tests unitaires
- **Mockito** pour le mocking des dependances
- **Testcontainers** pour les tests d'integration (PostgreSQL)
- Les tests couvrent la logique de gestion des patients, le flux ADT, et la recherche multi-criteres
