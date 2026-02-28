# Service Medical Technology

## Vue d'ensemble

Le service **Medical Technology** regroupe l'ensemble des activites de plateau technique : laboratoire d'analyses, imagerie medicale, gestion des equipements medicaux, et planification chirurgicale. Il assure le traitement des ordonnances de laboratoire et d'imagerie emises par le service Clinical, le suivi des specimens et resultats, ainsi que la maintenance des equipements.

| Propriete | Valeur |
|---|---|
| Port | `8084` |
| Base de donnees | `rofecare_medical_technology` (PostgreSQL, port `5435`) |
| Framework | Spring Boot 3.4.1, Java 21 |
| Architecture | Hexagonale / DDD |

## Architecture hexagonale

```
┌──────────────────────────────────────────────────────────────────┐
│                      infrastructure/                              │
│  ┌──────────────────────────┐  ┌───────────────────────────────┐ │
│  │  adapter/input/rest      │  │  adapter/output/persistence   │ │
│  │  - LaboratoryController  │  │  - LabOrderJpaRepository      │ │
│  │  - ImagingController     │  │  - ImagingOrderJpaRepository  │ │
│  │  - EquipmentController   │  │  - EquipmentJpaRepository     │ │
│  │  - SurgeryController     │  │  - SurgicalPlanJpaRepository  │ │
│  └──────────┬───────────────┘  └───────────────┬───────────────┘ │
│             │                                  │                 │
│  ┌──────────┴──────────────────────────────────┴───────────────┐ │
│  │  adapter/output/client                                      │ │
│  │  - ClinicalServiceClient (Feign)                            │ │
│  │  - PatientServiceClient (Feign)                             │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  config / security                                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                      application/                                 │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────────┐ │
│  │    port/      │ │     usecase/     │ │    dto / mapper       │ │
│  │  - InputPort  │ │ - LabOrderUC     │ │  - LabOrderDTO        │ │
│  │  - OutputPort │ │ - ImagingOrderUC │ │  - ImagingOrderDTO    │ │
│  │              │ │ - EquipmentUC    │ │  - EquipmentDTO       │ │
│  │              │ │ - SurgicalPlanUC │ │  - SurgicalPlanDTO    │ │
│  └──────────────┘ └──────────────────┘ └───────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  listener/                                                  │ │
│  │  - ClinicalEventListener                                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        domain/                                    │
│  ┌──────────────────┐ ┌────────────┐ ┌──────────┐ ┌──────────┐  │
│  │      model        │ │ repository │ │  service  │ │  event   │  │
│  │  - LabOrder       │ │            │ │           │ │          │  │
│  │  - LabResult      │ │            │ │           │ │          │  │
│  │  - Specimen       │ │            │ │           │ │          │  │
│  │  - ImagingOrder   │ │            │ │           │ │          │  │
│  │  - ImagingResult  │ │            │ │           │ │          │  │
│  │  - Equipment      │ │            │ │           │ │          │  │
│  │-MaintenanceSchedule│ │           │ │           │ │          │  │
│  │  - SurgicalPlan   │ │            │ │           │ │          │  │
│  │  - PreOpChecklist │ │            │ │           │ │          │  │
│  └──────────────────┘ └────────────┘ └──────────┘ └──────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

## Responsabilites

- **Gestion du laboratoire** : traitement des ordonnances de laboratoire, suivi des specimens (prelevement, transport, analyse), et publication des resultats
- **Imagerie medicale** : gestion des demandes d'imagerie, integration DICOM, et redaction des comptes rendus radiologiques
- **Gestion des equipements** : inventaire des dispositifs medicaux, planification et suivi de la maintenance preventive et corrective
- **Planification chirurgicale** : gestion des checklists pre-operatoires, planification du bloc operatoire
- **Gestion des resultats diagnostiques** : centralisation et diffusion des resultats d'examens
- **Controle qualite** : validation des resultats de laboratoire selon les normes de qualite

## Modele de domaine

### LabOrder
Ordonnance de laboratoire recue du service Clinical. Contient les examens demandes, le patient concerne, le praticien prescripteur, la priorite (normal, urgent, stat), et le statut de traitement.

### LabResult
Resultat d'un examen de laboratoire. Contient les valeurs mesurees, les unites, les plages de reference, le statut de validation (preliminaire, valide, corrige), et le biologiste validateur.

### Specimen
Echantillon biologique preleve pour analyse. Suivi du cycle de vie complet : prelevement, transport, reception au laboratoire, traitement, archivage. Identifie par un code-barres unique.

### ImagingOrder
Demande d'examen d'imagerie (radiographie, scanner, IRM, echographie). Contient la modalite, la region anatomique, l'indication clinique, et la priorite.

### ImagingResult
Resultat d'un examen d'imagerie. Comprend le compte rendu du radiologue, les references DICOM vers les images, et le statut de validation.

### Equipment
Dispositif medical enregistre dans l'inventaire. Contient les informations d'identification (numero de serie, modele), la localisation, le statut operationnel, et les dates de maintenance.

### MaintenanceSchedule
Planification de la maintenance d'un equipement. Definit la periodicite des interventions preventives, l'historique des maintenances correctives, et les prochaines echeances.

### SurgicalPlan
Plan pre-operatoire pour une intervention chirurgicale. Inclut le type d'intervention, l'equipe chirurgicale, la date prevue, les besoins en equipements, et la checklist pre-operatoire.

### PreOpChecklist
Liste de verification pre-operatoire. Permet de s'assurer que toutes les conditions sont remplies avant une intervention : consentement signe, resultats de laboratoire disponibles, bilan d'imagerie complet, jeune respecte.

## Endpoints API

### Laboratoire

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/medical-technology/laboratory/orders` | Lister les ordonnances de laboratoire | Oui (`LAB_READ`) |
| `POST` | `/api/medical-technology/laboratory/orders` | Creer une ordonnance de laboratoire | Oui (`LAB_CREATE`) |
| `GET` | `/api/medical-technology/laboratory/orders/{id}` | Obtenir les details d'une ordonnance | Oui (`LAB_READ`) |
| `PUT` | `/api/medical-technology/laboratory/orders/{id}/status` | Mettre a jour le statut d'une ordonnance | Oui (`LAB_UPDATE`) |
| `POST` | `/api/medical-technology/laboratory/results` | Enregistrer un resultat de laboratoire | Oui (`LAB_RESULT_CREATE`) |
| `GET` | `/api/medical-technology/laboratory/results/{id}` | Obtenir un resultat | Oui (`LAB_RESULT_READ`) |
| `PUT` | `/api/medical-technology/laboratory/results/{id}/validate` | Valider un resultat | Oui (`LAB_RESULT_VALIDATE`) |
| `POST` | `/api/medical-technology/laboratory/specimens` | Enregistrer un specimen | Oui (`SPECIMEN_CREATE`) |
| `PUT` | `/api/medical-technology/laboratory/specimens/{id}/status` | Mettre a jour le statut d'un specimen | Oui (`SPECIMEN_UPDATE`) |

### Imagerie medicale

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/medical-technology/imaging/orders` | Lister les demandes d'imagerie | Oui (`IMAGING_READ`) |
| `POST` | `/api/medical-technology/imaging/orders` | Creer une demande d'imagerie | Oui (`IMAGING_CREATE`) |
| `GET` | `/api/medical-technology/imaging/orders/{id}` | Obtenir les details d'une demande | Oui (`IMAGING_READ`) |
| `PUT` | `/api/medical-technology/imaging/orders/{id}/status` | Mettre a jour le statut | Oui (`IMAGING_UPDATE`) |
| `POST` | `/api/medical-technology/imaging/results` | Enregistrer un resultat d'imagerie | Oui (`IMAGING_RESULT_CREATE`) |
| `GET` | `/api/medical-technology/imaging/results/{id}` | Obtenir un resultat d'imagerie | Oui (`IMAGING_RESULT_READ`) |

### Equipements

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/medical-technology/equipment` | Lister les equipements | Oui (`EQUIPMENT_READ`) |
| `POST` | `/api/medical-technology/equipment` | Enregistrer un equipement | Oui (`EQUIPMENT_CREATE`) |
| `GET` | `/api/medical-technology/equipment/{id}` | Obtenir les details d'un equipement | Oui (`EQUIPMENT_READ`) |
| `PUT` | `/api/medical-technology/equipment/{id}` | Modifier un equipement | Oui (`EQUIPMENT_UPDATE`) |
| `GET` | `/api/medical-technology/equipment/{id}/maintenance` | Historique de maintenance | Oui (`MAINTENANCE_READ`) |
| `POST` | `/api/medical-technology/equipment/{id}/maintenance` | Planifier une maintenance | Oui (`MAINTENANCE_CREATE`) |

### Chirurgie

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/medical-technology/surgery/plans` | Lister les plans chirurgicaux | Oui (`SURGERY_READ`) |
| `POST` | `/api/medical-technology/surgery/plans` | Creer un plan chirurgical | Oui (`SURGERY_CREATE`) |
| `GET` | `/api/medical-technology/surgery/plans/{id}` | Obtenir les details d'un plan | Oui (`SURGERY_READ`) |
| `PUT` | `/api/medical-technology/surgery/plans/{id}` | Modifier un plan chirurgical | Oui (`SURGERY_UPDATE`) |
| `GET` | `/api/medical-technology/surgery/plans/{id}/checklist` | Obtenir la checklist pre-operatoire | Oui (`SURGERY_READ`) |
| `PUT` | `/api/medical-technology/surgery/plans/{id}/checklist` | Mettre a jour la checklist | Oui (`SURGERY_UPDATE`) |

## Evenements Kafka

### Evenements produits

| Topic | Evenement | Description | Payload |
|---|---|---|---|
| `medtech-events` | `laboratory-result-available` | Emis lorsqu'un resultat de laboratoire est valide | `labResultId`, `labOrderId`, `patientId`, `testType`, `status`, `timestamp` |
| `medtech-events` | `imaging-result-available` | Emis lorsqu'un resultat d'imagerie est disponible | `imagingResultId`, `imagingOrderId`, `patientId`, `modality`, `timestamp` |
| `medtech-events` | `equipment-maintenance-due` | Emis lorsqu'un equipement necessite une maintenance | `equipmentId`, `maintenanceType`, `dueDate`, `timestamp` |

### Evenements consommes

| Topic | Evenement | Description |
|---|---|---|
| `clinical-events` | `prescription-ordered` | Reception des ordonnances de laboratoire et d'imagerie emises par le service Clinical |

## Feign Clients

| Service cible | Usage |
|---|---|
| **Clinical** (`clinical-service`) | Recuperation des details des ordonnances, mise a jour du statut des prescriptions |
| **Patient** (`patient-service`) | Recuperation des informations du patient pour l'identification des specimens et la contextualisation des resultats |

## Configuration

```yaml
server:
  port: 8084

spring:
  datasource:
    url: jdbc:postgresql://localhost:5435/rofecare_medical_technology
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
      group-id: medical-technology-service
      auto-offset-reset: earliest

feign:
  client:
    config:
      clinical-service:
        url: http://localhost:8083
        connect-timeout: 5000
        read-timeout: 5000
      patient-service:
        url: http://localhost:8082
        connect-timeout: 5000
        read-timeout: 5000

jwt:
  secret: ${JWT_SECRET}
```

## Schema de base de donnees (tables principales)

| Table | Description |
|---|---|
| `lab_orders` | Ordonnances de laboratoire |
| `lab_results` | Resultats d'examens de laboratoire |
| `specimens` | Specimens biologiques avec suivi |
| `imaging_orders` | Demandes d'examens d'imagerie |
| `imaging_results` | Resultats et comptes rendus d'imagerie |
| `equipment` | Inventaire des equipements medicaux |
| `maintenance_schedules` | Planification de la maintenance |
| `maintenance_history` | Historique des interventions de maintenance |
| `surgical_plans` | Plans chirurgicaux |
| `pre_op_checklists` | Checklists pre-operatoires |

## Tests

Le service utilise :

- **JUnit 5** pour les tests unitaires
- **Mockito** pour le mocking des dependances
- **Testcontainers** pour les tests d'integration (PostgreSQL)
- Les tests couvrent les workflows de laboratoire (ordonnance -> specimen -> resultat -> validation), le cycle de vie des demandes d'imagerie, et la gestion des equipements
