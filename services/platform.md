# Service Platform

> **Port** : 8087
> **Base de donnees** : `rofecare_platform` (PostgreSQL, port 5438)
> **Module Maven** : `rofecare-platform`
> **Package racine** : `com.rofecare.platform`
> **Tests unitaires** : 128

## Vue d'ensemble

Le service Platform fournit les fonctionnalites transversales de la plateforme Rofecare : notifications multi-canal, journalisation d'audit, gestion documentaire, planification des rendez-vous, administration systeme et generation de rapports. Il agit comme consommateur central des evenements Kafka produits par tous les autres services, ce qui en fait le hub d'observation de l'activite du systeme.

## Architecture

```
com.rofecare.platform/
├── domain/
│   ├── model/            # Notification, AuditLog, Document, Appointment, etc.
│   ├── port/
│   │   ├── in/           # NotificationUseCase, AuditUseCase, DocumentUseCase
│   │   └── out/          # NotificationRepository, AuditLogRepository, etc.
│   ├── service/          # NotificationDomainService, SchedulingDomainService
│   └── event/            # NotificationSentEvent, AppointmentCreatedEvent
├── application/
│   ├── service/          # NotificationApplicationService, AuditApplicationService
│   ├── dto/              # Request/Response DTOs
│   └── mapper/           # Domain <-> DTO mappers
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/      # REST controllers
    │   └── out/
    │       ├── persistence/  # JPA entities, repositories
    │       ├── kafka/        # Central event consumer
    │       ├── email/        # Email sender adapter (SMTP)
    │       ├── sms/          # SMS sender adapter
    │       └── storage/      # Document storage adapter (S3/local)
    └── config/           # PlatformSecurityConfig, StorageConfig
```

## Modeles du domaine

| Modele | Description |
|--------|-------------|
| `Notification` | Notification envoyee a un utilisateur. Contient le canal (EMAIL, SMS, PUSH, IN_APP), le statut de livraison, le contenu et le destinataire. |
| `NotificationTemplate` | Template reutilisable pour les notifications. Supporte les variables de substitution (nom du patient, date du rendez-vous, etc.). |
| `AuditLog` | Entree de journal d'audit. Enregistre chaque action systeme : qui a fait quoi, quand, sur quelle ressource, avec les valeurs avant/apres. |
| `Document` | Document stocke dans le systeme : resultats de laboratoire, imagerie, comptes-rendus, pieces jointes. Metadata : type, taille, format, proprietaire. |
| `Appointment` | Rendez-vous planifie entre un patient et un praticien. Gere les creneaux horaires, les confirmations et les annulations. |
| `Schedule` | Planning d'un praticien ou d'une ressource. Definit les creneaux disponibles, les jours de travail et les exceptions. |
| `SystemSetting` | Parametre de configuration systeme. Permet la personnalisation de l'application sans redeploiement (nom de l'etablissement, devise, fuseau horaire, etc.). |
| `Report` | Rapport genere a la demande ou de maniere programmee. Supporte les formats PDF et Excel. Contient les parametres de generation et le fichier resultat. |

## Endpoints REST

### Notifications

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/platform/notifications` | Envoyer une notification |
| `GET` | `/api/platform/notifications` | Lister les notifications avec filtres |
| `GET` | `/api/platform/notifications/{id}` | Recuperer une notification par ID |
| `PATCH` | `/api/platform/notifications/{id}/read` | Marquer une notification comme lue |
| `GET` | `/api/platform/notifications/unread/count` | Compter les notifications non lues |
| `POST` | `/api/platform/notifications/templates` | Creer un template de notification |
| `GET` | `/api/platform/notifications/templates` | Lister les templates |

### Journaux d'audit

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/platform/audit-logs` | Lister les entrees d'audit avec filtres (date, utilisateur, action, ressource) |
| `GET` | `/api/platform/audit-logs/{id}` | Recuperer une entree d'audit par ID |
| `GET` | `/api/platform/audit-logs/user/{userId}` | Recuperer les actions d'un utilisateur |
| `GET` | `/api/platform/audit-logs/resource/{resourceType}/{resourceId}` | Historique d'une ressource specifique |

### Documents

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/platform/documents` | Telecharger (upload) un document |
| `GET` | `/api/platform/documents` | Lister les documents avec filtres |
| `GET` | `/api/platform/documents/{id}` | Recuperer les metadonnees d'un document |
| `GET` | `/api/platform/documents/{id}/download` | Telecharger le fichier |
| `DELETE` | `/api/platform/documents/{id}` | Supprimer un document |

### Rendez-vous

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/platform/appointments` | Creer un rendez-vous |
| `GET` | `/api/platform/appointments` | Lister les rendez-vous avec filtres |
| `GET` | `/api/platform/appointments/{id}` | Recuperer un rendez-vous par ID |
| `PUT` | `/api/platform/appointments/{id}` | Modifier un rendez-vous |
| `PATCH` | `/api/platform/appointments/{id}/cancel` | Annuler un rendez-vous |
| `PATCH` | `/api/platform/appointments/{id}/confirm` | Confirmer un rendez-vous |
| `GET` | `/api/platform/appointments/available-slots` | Recuperer les creneaux disponibles |

### Rapports

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/platform/reports/generate` | Generer un rapport |
| `GET` | `/api/platform/reports` | Lister les rapports generes |
| `GET` | `/api/platform/reports/{id}` | Telecharger un rapport |
| `GET` | `/api/platform/reports/templates` | Lister les modeles de rapports disponibles |

## Evenements Kafka

### Evenements produits

| Evenement | Topic | Description |
|-----------|-------|-------------|
| `notification-sent` | `platform.notification.sent` | Emis apres l'envoi reussi d'une notification. Contient le canal, le destinataire et le statut. |
| `appointment-created` | `platform.appointment.created` | Emis lors de la creation d'un rendez-vous. Contient patient, praticien, date et heure. |
| `report-generated` | `platform.report.generated` | Emis lorsqu'un rapport est genere avec succes. |

### Evenements consommes

Le service Platform est le **consommateur central** de l'ecosysteme Rofecare. Il ecoute les evenements majeurs de tous les services pour :

- **Audit** : Enregistrer toute action significative dans le journal d'audit
- **Notifications** : Declencher des notifications contextuelles aux utilisateurs concernes
- **Dashboards** : Agreger les donnees pour les tableaux de bord en temps reel

| Source | Evenements consommes | Reaction |
|--------|---------------------|----------|
| Service Identity | `user-created`, `user-updated`, `role-changed` | Audit log + notification a l'administrateur |
| Service Patient | `patient-registered`, `patient-updated` | Audit log |
| Service Clinical | `consultation-completed`, `patient-admitted`, `patient-discharged` | Audit log + notification au patient + mise a jour dashboard |
| Service MedTech | `lab-result-available`, `imaging-completed` | Notification au medecin prescripteur |
| Service Pharmacy | `medication-dispensed`, `stock-alert` | Audit log + notification pharmacien (stock alert) |
| Service Finance | `invoice-created`, `payment-received`, `insurance-claim-processed` | Audit log + notification au patient |

## Configuration

```yaml
# application.yml
server:
  port: 8087

spring:
  datasource:
    url: jdbc:postgresql://localhost:5438/rofecare_platform
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate

  kafka:
    bootstrap-servers: ${KAFKA_SERVERS:localhost:9092}
    consumer:
      group-id: platform-service
      auto-offset-reset: earliest

  mail:
    host: ${SMTP_HOST:localhost}
    port: ${SMTP_PORT:587}
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}

  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 50MB

storage:
  type: ${STORAGE_TYPE:local}          # local | s3
  local:
    base-path: ${STORAGE_PATH:/data/documents}
  s3:
    bucket: ${S3_BUCKET:rofecare-documents}
    region: ${S3_REGION:us-east-1}

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}
```

## Decisions architecturales

1. **Consommateur central d'evenements** : Le service Platform consomme les evenements de tous les autres services pour centraliser l'audit et les notifications, evitant ainsi la duplication de logique d'audit dans chaque service.

2. **Notifications multi-canal** : Le systeme de templates avec variables de substitution permet de gerer les notifications par email, SMS, push et in-app depuis un point unique, avec fallback configurable entre canaux.

3. **Stockage de documents abstrait** : L'adapter de stockage supporte le stockage local et Amazon S3, permettant une transition facile vers le cloud sans modification du code applicatif.

4. **Gestion des rendez-vous avec creneaux** : Le systeme de `Schedule` et `Appointment` gere la disponibilite des praticiens avec detection des conflits horaires, evitant les doubles reservations.

5. **Points d'integration AI/ML** : L'architecture prevoit des points d'extension pour l'integration future de fonctionnalites d'intelligence artificielle (aide au diagnostic, prediction de charge, etc.).

6. **Generation de rapports asynchrone** : Les rapports complexes sont generes de maniere asynchrone, avec notification a l'utilisateur lorsque le rapport est pret, evitant les timeouts sur les requetes longues.
