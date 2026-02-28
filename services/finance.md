# Service Finance

> **Port** : 8086
> **Base de donnees** : `rofecare_finance` (PostgreSQL, port 5437)
> **Module Maven** : `rofecare-finance`
> **Package racine** : `com.rofecare.finance`
> **Tests unitaires** : 264

## Vue d'ensemble

Le service Finance gere l'ensemble du cycle financier de l'etablissement de sante : facturation, encaissement, gestion des reclamations d'assurance, tarification et reporting financier. C'est le deuxieme plus grand service en termes de couverture de tests (264 tests unitaires), refletant la complexite et la criticite des operations financieres.

Ce service suit l'architecture hexagonale (Ports & Adapters) avec une separation stricte entre le domaine metier, la couche applicative et l'infrastructure.

## Architecture

```
com.rofecare.finance/
├── domain/
│   ├── model/            # Invoice, Payment, InsuranceClaim, FeeSchedule, etc.
│   ├── port/
│   │   ├── in/           # InvoiceUseCase, PaymentUseCase, InsuranceClaimUseCase
│   │   └── out/          # InvoiceRepository, PaymentRepository, etc.
│   ├── service/          # InvoiceDomainService, PaymentDomainService
│   └── event/            # InvoiceCreatedEvent, PaymentReceivedEvent
├── application/
│   ├── service/          # InvoiceApplicationService, PaymentApplicationService
│   ├── dto/              # Request/Response DTOs
│   └── mapper/           # Domain <-> DTO mappers
└── infrastructure/
    ├── adapter/
    │   ├── in/
    │   │   └── web/      # REST controllers
    │   └── out/
    │       ├── persistence/  # JPA entities, repositories
    │       ├── feign/        # PatientFeignClient, ClinicalFeignClient
    │       └── kafka/        # Event publishers & consumers
    └── config/           # FinanceSecurityConfig, KafkaConfig
```

## Modeles du domaine

| Modele | Description |
|--------|-------------|
| `Invoice` | Facture liee a un patient et une visite. Contient les lignes de facturation, le statut (DRAFT, SENT, PAID, CANCELLED), le montant total et les references d'assurance. |
| `InvoiceLine` | Ligne individuelle d'une facture : acte medical, medicament, sejour, etc. Referencee par code tarifaire. |
| `Payment` | Paiement recu. Supporte plusieurs methodes (especes, carte, mobile money, assurance). Lie a une ou plusieurs factures. |
| `PaymentMethod` | Methode de paiement configuree : CASH, CARD, MOBILE_MONEY, INSURANCE, BANK_TRANSFER. |
| `InsuranceClaim` | Reclamation d'assurance soumise a un organisme payeur. Suit le cycle : DRAFT -> SUBMITTED -> UNDER_REVIEW -> APPROVED/REJECTED -> RECONCILED. |
| `FeeSchedule` | Grille tarifaire definissant le prix des actes, medicaments et services. Peut varier selon le type de patient ou le contrat d'assurance. |
| `Account` | Compte comptable pour le suivi des revenus, depenses et creances. |
| `TaxRate` | Taux de taxe applicable aux lignes de facturation selon la categorie de service. |
| `FinancialReport` | Rapport financier genere (revenus, depenses, creances). Peut etre exporte en PDF ou Excel. |

## Endpoints REST

### Factures

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/finance/invoices` | Creer une nouvelle facture |
| `GET` | `/api/finance/invoices` | Lister les factures avec filtres et pagination |
| `GET` | `/api/finance/invoices/{id}` | Recuperer une facture par ID |
| `PUT` | `/api/finance/invoices/{id}` | Mettre a jour une facture |
| `PATCH` | `/api/finance/invoices/{id}/cancel` | Annuler une facture |
| `POST` | `/api/finance/invoices/{id}/send` | Envoyer une facture au patient/assureur |
| `GET` | `/api/finance/invoices/{id}/lines` | Lister les lignes d'une facture |

### Paiements

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/finance/payments` | Enregistrer un paiement |
| `GET` | `/api/finance/payments` | Lister les paiements avec filtres |
| `GET` | `/api/finance/payments/{id}` | Recuperer un paiement par ID |
| `POST` | `/api/finance/payments/{id}/refund` | Rembourser un paiement |

### Reclamations d'assurance

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `POST` | `/api/finance/insurance-claims` | Soumettre une reclamation |
| `GET` | `/api/finance/insurance-claims` | Lister les reclamations |
| `GET` | `/api/finance/insurance-claims/{id}` | Recuperer une reclamation par ID |
| `PATCH` | `/api/finance/insurance-claims/{id}/status` | Mettre a jour le statut d'une reclamation |
| `POST` | `/api/finance/insurance-claims/reconcile` | Reconcilier les reclamations avec les paiements recus |

### Rapports financiers

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/finance/reports` | Lister les rapports disponibles |
| `POST` | `/api/finance/reports/generate` | Generer un rapport financier |
| `GET` | `/api/finance/reports/{id}` | Telecharger un rapport |

### Grilles tarifaires

| Methode | Endpoint | Description |
|---------|----------|-------------|
| `GET` | `/api/finance/fee-schedules` | Lister les grilles tarifaires |
| `POST` | `/api/finance/fee-schedules` | Creer une grille tarifaire |
| `PUT` | `/api/finance/fee-schedules/{id}` | Mettre a jour une grille tarifaire |
| `GET` | `/api/finance/fee-schedules/{id}/items` | Lister les elements d'une grille |

## Evenements Kafka

### Evenements produits

| Evenement | Topic | Description |
|-----------|-------|-------------|
| `invoice-created` | `finance.invoice.created` | Emis lors de la creation d'une facture. Contient l'ID facture, patient, montant total. |
| `payment-received` | `finance.payment.received` | Emis lors de l'enregistrement d'un paiement. Contient l'ID paiement, facture, montant, methode. |
| `insurance-claim-submitted` | `finance.insurance-claim.submitted` | Emis lors de la soumission d'une reclamation a l'assureur. |
| `insurance-claim-processed` | `finance.insurance-claim.processed` | Emis lorsqu'une reclamation est approuvee ou rejetee. |

### Evenements consommes

| Evenement | Source | Reaction |
|-----------|--------|----------|
| `consultation-completed` | Service Clinical | Creation automatique d'une facture brouillon avec les actes realises. |
| `medication-dispensed` | Service Pharmacy | Ajout des lignes de medicaments a la facture du patient. |
| `patient-admitted` | Service Clinical | Creation d'un compte patient pour le suivi des frais d'hospitalisation. |

## Clients Feign

| Client | Service cible | Utilisation |
|--------|---------------|-------------|
| `PatientFeignClient` | Service Patient | Recuperer les informations d'assurance du patient, son adresse de facturation et ses contacts. |
| `ClinicalFeignClient` | Service Clinical | Recuperer les procedures et actes medicaux realises pour generer les lignes de facturation. |

## Configuration

```yaml
# application.yml
server:
  port: 8086

spring:
  datasource:
    url: jdbc:postgresql://localhost:5437/rofecare_finance
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: public

  kafka:
    bootstrap-servers: ${KAFKA_SERVERS:localhost:9092}
    consumer:
      group-id: finance-service
      auto-offset-reset: earliest

eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_URL:http://localhost:8761/eureka}

feign:
  client:
    config:
      patient-service:
        url: ${PATIENT_SERVICE_URL:http://localhost:8083}
      clinical-service:
        url: ${CLINICAL_SERVICE_URL:http://localhost:8084}
```

## Decisions architecturales

1. **Facturation event-driven** : Les factures sont creees automatiquement a partir des evenements `consultation-completed` et `medication-dispensed`, evitant le couplage direct avec les services cliniques et pharmaceutiques.

2. **Multi-methode de paiement** : Le systeme supporte simultanement especes, carte bancaire, mobile money et paiement par assurance, avec possibilite de paiement partiel et multi-methode sur une meme facture.

3. **Cycle de vie des reclamations d'assurance** : Les reclamations suivent un workflow rigoureux (DRAFT -> SUBMITTED -> UNDER_REVIEW -> APPROVED/REJECTED -> RECONCILED) avec audit trail complet.

4. **Grilles tarifaires flexibles** : Le systeme de `FeeSchedule` permet de definir des tarifications differentes selon le type de patient, le contrat d'assurance ou la periode, offrant une grande flexibilite pour les etablissements multi-conventions.

5. **Separation comptable** : Le modele `Account` permet un suivi comptable conforme aux normes, avec distinction entre revenus, creances et depenses, facilitant l'integration future avec des systemes comptables externes.

6. **Calcul des taxes** : Les taux de taxe sont geres de maniere flexible via `TaxRate`, permettant l'adaptation aux reglementations fiscales locales sans modification du code.
