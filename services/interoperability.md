# Service Interoperability

> **Port** : 8088
> **Base de donnees** : `rofecare_interoperability` (PostgreSQL, port 5439)
> **Module Maven** : `rofecare-interoperability`
> **Package racine** : `com.rofecare.interoperability`
> **Tests unitaires** : 95

## Vue d'ensemble

Le service Interoperability assure la connectivite de Rofecare avec les systemes externes en implementant les standards internationaux de sante : HL7 FHIR R4, HL7 v2 et DICOM. Il gere la transformation des donnees entre le modele interne de Rofecare et les formats standardises, les webhooks sortants, et maintient une piste d'audit complete de toutes les integrations.

Ce service est essentiel pour l'integration dans un ecosysteme de sante plus large, permettant l'echange de donnees avec d'autres systemes d'information hospitaliers, les laboratoires externes, les systemes d'imagerie medicale et les plateformes nationales de sante.

## Architecture

```
com.rofecare.interoperability/
├── domain/
│   ├── model/            # FhirResource, Hl7Message, DicomStudy, etc.
│   ├── port/
│   │   ├── in/           # FhirUseCase, Hl7UseCase, DicomUseCase
│   │   └── out/          # FhirRepository, ExternalSystemRepository, etc.
│   ├── service/          # FhirMappingService, Hl7ProcessingService
│   └── event/            # IntegrationCompletedEvent
├── application/
│   ├── service/          # FhirApplicationService, Hl7ApplicationService
│   ├── dto/              # Request/Response DTOs
│   └── mapper/           # Rofecare <-> FHIR/HL7 mappers
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/      # REST controllers (FHIR endpoints, HL7 receivers)
    │   └── out/
    │       ├── persistence/  # JPA entities, repositories
    │       ├── fhir/         # HAPI FHIR client adapter
    │       ├── hl7/          # HL7 v2 message parser/builder
    │       ├── dicom/        # DICOM client adapter
    │       └── webhook/      # Webhook dispatcher
    └── config/           # InteroperabilitySecurityConfig, FhirConfig
```

## Modeles du domaine

| Modele | Description |
|--------|-------------|
| `FhirResource` | Representation interne d'une ressource FHIR. Contient le type de ressource, l'identifiant FHIR, le contenu JSON/XML et les metadonnees de mapping. |
| `Hl7Message` | Message HL7 v2 recu ou envoye. Contient le type de message (ADT, ORM, ORU, etc.), le contenu brut et le statut de traitement. |
| `DicomStudy` | Reference a une etude DICOM. Contient les identifiants DICOM (StudyInstanceUID), le type de modalite, le nombre d'images et le lien vers le serveur PACS. |
| `ExternalSystem` | Systeme externe enregistre. Contient l'URL, les credentials, le protocole supporte (FHIR, HL7v2, REST), le statut de connexion et la derniere synchronisation. |
| `Webhook` | Configuration de webhook sortant. Definit l'URL cible, les evenements declencheurs, le format de payload et la politique de retry. |
| `IntegrationMapping` | Regle de mapping entre le modele Rofecare et un format externe. Permet la configuration des transformations sans modification du code. |
| `DataTransformation` | Transformation de donnees appliquee lors d'un echange. Enregistre les donnees source, le resultat de la transformation et les eventuelles erreurs. |

## Endpoints REST

### FHIR R4

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/interoperability/fhir/Patient` | Rechercher des patients au format FHIR |
| `GET` | `/api/interoperability/fhir/Patient/{id}` | Recuperer un patient au format FHIR |
| `GET` | `/api/interoperability/fhir/Encounter` | Rechercher des rencontres au format FHIR |
| `GET` | `/api/interoperability/fhir/Encounter/{id}` | Recuperer une rencontre au format FHIR |
| `GET` | `/api/interoperability/fhir/Observation` | Rechercher des observations au format FHIR |
| `GET` | `/api/interoperability/fhir/DiagnosticReport` | Rechercher des rapports diagnostiques |
| `GET` | `/api/interoperability/fhir/MedicationRequest` | Rechercher des prescriptions au format FHIR |
| `GET` | `/api/interoperability/fhir/Practitioner` | Rechercher des praticiens au format FHIR |
| `POST` | `/api/interoperability/fhir/Bundle` | Soumettre un bundle FHIR (transaction) |
| `GET` | `/api/interoperability/fhir/metadata` | Capability Statement (conformance) |

### HL7 v2

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/interoperability/hl7/messages` | Recevoir un message HL7 v2 |
| `GET` | `/api/interoperability/hl7/messages` | Lister les messages HL7 traites |
| `GET` | `/api/interoperability/hl7/messages/{id}` | Recuperer un message HL7 par ID |
| `GET` | `/api/interoperability/hl7/messages/{id}/ack` | Recuperer l'accusé de reception (ACK) |

### DICOM

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/interoperability/dicom/studies` | Rechercher des etudes DICOM (QIDO-RS) |
| `GET` | `/api/interoperability/dicom/studies/{studyUID}` | Recuperer les metadonnees d'une etude |
| `GET` | `/api/interoperability/dicom/studies/{studyUID}/series` | Lister les series d'une etude |
| `POST` | `/api/interoperability/dicom/studies` | Stocker une etude DICOM (STOW-RS) |

### Webhooks

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/interoperability/webhooks` | Creer un webhook |
| `GET` | `/api/interoperability/webhooks` | Lister les webhooks configures |
| `GET` | `/api/interoperability/webhooks/{id}` | Recuperer un webhook par ID |
| `PUT` | `/api/interoperability/webhooks/{id}` | Modifier un webhook |
| `DELETE` | `/api/interoperability/webhooks/{id}` | Supprimer un webhook |
| `POST` | `/api/interoperability/webhooks/{id}/test` | Tester un webhook (envoi d'un payload de test) |
| `GET` | `/api/interoperability/webhooks/{id}/deliveries` | Historique des livraisons d'un webhook |

### Systemes externes

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/interoperability/systems` | Enregistrer un systeme externe |
| `GET` | `/api/interoperability/systems` | Lister les systemes externes |
| `GET` | `/api/interoperability/systems/{id}` | Recuperer un systeme externe par ID |
| `PUT` | `/api/interoperability/systems/{id}` | Modifier un systeme externe |
| `POST` | `/api/interoperability/systems/{id}/test-connection` | Tester la connexion |
| `GET` | `/api/interoperability/systems/{id}/logs` | Consulter les logs d'integration |

## Ressources FHIR supportees

| Ressource FHIR | Mapping Rofecare | Operations |
|-----------------|------------------|------------|
| `Patient` | Service Patient -> FHIR Patient | Read, Search, Create |
| `Encounter` | Consultation/Admission -> FHIR Encounter | Read, Search |
| `Observation` | Resultats laboratoire -> FHIR Observation | Read, Search, Create |
| `DiagnosticReport` | Rapports medicaux -> FHIR DiagnosticReport | Read, Search |
| `MedicationRequest` | Prescriptions -> FHIR MedicationRequest | Read, Search |
| `Practitioner` | Utilisateurs medecins -> FHIR Practitioner | Read, Search |

## Evenements Kafka

Le service Interoperability ne produit pas d'evenements Kafka de maniere systematique car il agit principalement comme un **adaptateur** entre Rofecare et le monde exterieur. Cependant, il consomme les evenements internes pour declencher des notifications webhook vers les systemes externes enregistres.

### Consommation d'evenements

Lorsqu'un evenement interne correspond a un webhook configure, le service transforme l'evenement au format attendu par le systeme externe et l'envoie via HTTP.

## Configuration

```yaml
# application.yml
server:
  port: 8088

spring:
  datasource:
    url: jdbc:postgresql://localhost:5439/rofecare_interoperability
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate

  kafka:
    bootstrap-servers: ${KAFKA_SERVERS:localhost:9092}
    consumer:
      group-id: interoperability-service
      auto-offset-reset: earliest

fhir:
  server:
    base-url: ${FHIR_BASE_URL:http://localhost:8088/api/interoperability/fhir}
    version: R4
    default-format: json         # json | xml
    pagination:
      default-count: 20
      max-count: 100

hl7:
  v2:
    encoding: UTF-8
    version: "2.5.1"
    sending-application: ROFECARE
    sending-facility: ${FACILITY_NAME:ROFECARE_HIS}

dicom:
  pacs:
    url: ${PACS_URL:http://localhost:8042}
    aet: ${PACS_AET:ROFECARE}

webhook:
  retry:
    max-attempts: 3
    backoff-multiplier: 2
    initial-delay-ms: 1000
  timeout:
    connect-ms: 5000
    read-ms: 30000

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
```

## Decisions architecturales

1. **HAPI FHIR comme moteur** : L'implementation FHIR utilise la librairie HAPI FHIR pour le parsing, la validation et la serialisation des ressources, garantissant la conformite au standard R4.

2. **Mapping configurable** : Les regles de transformation entre le modele Rofecare et les formats externes sont stockees dans `IntegrationMapping`, permettant l'ajout de nouveaux systemes externes sans modification du code.

3. **Webhooks avec retry** : Les webhooks sortants utilisent une politique de retry avec backoff exponentiel (3 tentatives par defaut), garantissant la fiabilite de la livraison des notifications aux systemes externes.

4. **DICOMweb** : L'integration DICOM utilise le protocole DICOMweb (QIDO-RS, WADO-RS, STOW-RS) plutot que le protocole DIMSE traditionnel, offrant une integration plus simple via HTTP/REST.

5. **Piste d'audit des integrations** : Chaque echange avec un systeme externe est enregistre dans `DataTransformation`, permettant le diagnostic des problemes d'interoperabilite et la tracabilite complete des echanges.

6. **Capability Statement** : L'endpoint `/fhir/metadata` expose automatiquement les capacites FHIR du serveur, facilitant la decouverte et l'integration par les systemes clients.
