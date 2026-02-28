# Service Pharmacy

## Vue d'ensemble

Le service **Pharmacy** gere l'ensemble des activites pharmaceutiques de l'etablissement : du catalogue de medicaments a la dispensation, en passant par la validation des prescriptions, le controle des interactions medicamenteuses, et la gestion des stocks. Il assure egalement le suivi des substances controlees conformement aux reglementations en vigueur.

| Propriete | Valeur |
|---|---|
| Port | `8085` |
| Base de donnees | `rofecare_pharmacy` (PostgreSQL, port `5436`) |
| Framework | Spring Boot 3.4.1, Java 21 |
| Architecture | Hexagonale / DDD |
| Tests unitaires | 111 |

## Architecture hexagonale

```
┌──────────────────────────────────────────────────────────────────┐
│                      infrastructure/                              │
│  ┌──────────────────────────┐  ┌───────────────────────────────┐ │
│  │  adapter/input/rest      │  │  adapter/output/persistence   │ │
│  │  - MedicationController  │  │  - MedicationJpaRepository    │ │
│  │  - DispensingController  │  │  - DispensingJpaRepository    │ │
│  │  - InventoryController   │  │  - StockItemJpaRepository     │ │
│  │  - OrderController       │  │  - PharmacyOrderJpaRepository │ │
│  │  - InteractionController │  │  - DrugInteractionJpaRepo     │ │
│  └──────────┬───────────────┘  └───────────────┬───────────────┘ │
│             │                                  │                 │
│  ┌──────────┴──────────────────────────────────┴───────────────┐ │
│  │  adapter/output/client                                      │ │
│  │  - ClinicalServiceClient (Feign)                            │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  config / security                                          │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                      application/                                 │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────────┐ │
│  │    port/      │ │     usecase/     │ │    dto / mapper       │ │
│  │  - InputPort  │ │ - DispensingUC   │ │  - MedicationDTO      │ │
│  │  - OutputPort │ │ - InventoryUC    │ │  - DispensingDTO      │ │
│  │              │ │ - InteractionUC  │ │  - StockItemDTO       │ │
│  │              │ │ - OrderUC        │ │  - PharmacyOrderDTO   │ │
│  └──────────────┘ └──────────────────┘ └───────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  listener/                                                  │ │
│  │  - ClinicalEventListener (prescription-ordered)             │ │
│  └─────────────────────────────────────────────────────────────┘ │
├──────────────────────────────────────────────────────────────────┤
│                        domain/                                    │
│  ┌──────────────────────┐ ┌────────────┐ ┌─────────┐ ┌────────┐ │
│  │       model           │ │ repository │ │ service  │ │ event  │ │
│  │  - Medication         │ │            │ │          │ │        │ │
│  │  - Prescription       │ │            │ │          │ │        │ │
│  │  - Dispensing         │ │            │ │          │ │        │ │
│  │  - StockItem          │ │            │ │          │ │        │ │
│  │  - DrugInteraction    │ │            │ │          │ │        │ │
│  │  - PharmacyOrder      │ │            │ │          │ │        │ │
│  │  - Supplier           │ │            │ │          │ │        │ │
│  │-ControlledSubstanceLog│ │            │ │          │ │        │ │
│  └──────────────────────┘ └────────────┘ └─────────┘ └────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

## Responsabilites

- **Gestion du catalogue de medicaments** : reference complete des medicaments disponibles (denomination, dosage, forme pharmaceutique, classification ATC)
- **Traitement et validation des prescriptions** : reception, verification et validation pharmaceutique des ordonnances
- **Workflow de dispensation** : preparation, verification, et remise des medicaments au patient
- **Verification des interactions medicamenteuses** : detection automatique des interactions entre les medicaments prescrits a un meme patient
- **Gestion des stocks** : suivi des niveaux de stock, alertes de reapprovisionnement, gestion des lots et dates de peremption
- **Suivi des substances controlees** : traçabilite stricte des stupefiants et substances soumises a reglementation
- **Gestion des commandes pharmaceutiques** : commandes aux fournisseurs, reception des livraisons
- **Gestion des fournisseurs** : repertoire des fournisseurs, contrats et conditions

## Modele de domaine

### Medication
Medicament reference dans le catalogue. Contient la denomination commune internationale (DCI), le nom commercial, le dosage, la forme pharmaceutique, la classification ATC, le code CIP, et les conditions de stockage.

### Prescription
Copie locale de la prescription emise par le service Clinical. Utilisee pour le processus de validation pharmaceutique et de dispensation. Contient les details de posologie, duree, et frequence.

### Dispensing
Enregistrement de la dispensation d'un medicament a un patient. Contient le medicament dispense, la quantite, le lot, la date de peremption, le pharmacien responsable, et le statut (prepare, verifie, dispense).

### StockItem
Article en stock dans la pharmacie. Represente un lot specifique d'un medicament avec la quantite disponible, le numero de lot, la date de peremption, et le seuil de reapprovisionnement.

### DrugInteraction
Reference d'interaction entre deux medicaments. Contient la severite (mineure, moderee, majeure, contre-indication), la description de l'interaction, et la conduite a tenir.

### PharmacyOrder
Commande de medicaments aupres d'un fournisseur. Contient les articles commandes, les quantites, le fournisseur, la date de commande, la date de livraison prevue, et le statut.

### Supplier
Fournisseur de medicaments et produits pharmaceutiques. Contient les coordonnees, les conditions commerciales, et les delais de livraison habituels.

### ControlledSubstanceLog
Journal de traçabilite des substances controlees. Enregistre chaque mouvement (entree, sortie, destruction) avec l'identite de l'operateur, la quantite, et le solde restant.

## Endpoints API

### Medicaments

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/pharmacy/medications` | Lister les medicaments du catalogue | Oui (`MEDICATION_READ`) |
| `POST` | `/api/pharmacy/medications` | Ajouter un medicament au catalogue | Oui (`MEDICATION_CREATE`) |
| `GET` | `/api/pharmacy/medications/{id}` | Obtenir les details d'un medicament | Oui (`MEDICATION_READ`) |
| `PUT` | `/api/pharmacy/medications/{id}` | Modifier un medicament | Oui (`MEDICATION_UPDATE`) |
| `GET` | `/api/pharmacy/medications/search` | Rechercher un medicament (nom, DCI, ATC) | Oui (`MEDICATION_READ`) |

### Dispensation

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/pharmacy/dispensing` | Lister les dispensations | Oui (`DISPENSING_READ`) |
| `POST` | `/api/pharmacy/dispensing` | Enregistrer une dispensation | Oui (`DISPENSING_CREATE`) |
| `GET` | `/api/pharmacy/dispensing/{id}` | Obtenir les details d'une dispensation | Oui (`DISPENSING_READ`) |
| `PUT` | `/api/pharmacy/dispensing/{id}/status` | Mettre a jour le statut de dispensation | Oui (`DISPENSING_UPDATE`) |
| `GET` | `/api/pharmacy/dispensing/patient/{patientId}` | Historique de dispensation d'un patient | Oui (`DISPENSING_READ`) |

### Inventaire

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/pharmacy/inventory` | Lister les articles en stock | Oui (`INVENTORY_READ`) |
| `POST` | `/api/pharmacy/inventory` | Ajouter un article au stock | Oui (`INVENTORY_CREATE`) |
| `GET` | `/api/pharmacy/inventory/{id}` | Obtenir les details d'un article | Oui (`INVENTORY_READ`) |
| `PUT` | `/api/pharmacy/inventory/{id}` | Modifier un article en stock | Oui (`INVENTORY_UPDATE`) |
| `GET` | `/api/pharmacy/inventory/low-stock` | Lister les articles en rupture ou bas stock | Oui (`INVENTORY_READ`) |
| `GET` | `/api/pharmacy/inventory/expiring` | Lister les articles proches de la peremption | Oui (`INVENTORY_READ`) |

### Commandes

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `GET` | `/api/pharmacy/orders` | Lister les commandes | Oui (`ORDER_READ`) |
| `POST` | `/api/pharmacy/orders` | Creer une commande fournisseur | Oui (`ORDER_CREATE`) |
| `GET` | `/api/pharmacy/orders/{id}` | Obtenir les details d'une commande | Oui (`ORDER_READ`) |
| `PUT` | `/api/pharmacy/orders/{id}/status` | Mettre a jour le statut d'une commande | Oui (`ORDER_UPDATE`) |
| `POST` | `/api/pharmacy/orders/{id}/receive` | Enregistrer la reception d'une commande | Oui (`ORDER_UPDATE`) |

### Interactions medicamenteuses

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `POST` | `/api/pharmacy/interactions/check` | Verifier les interactions pour une liste de medicaments | Oui (`INTERACTION_READ`) |
| `GET` | `/api/pharmacy/interactions/{medicationId}` | Lister les interactions connues d'un medicament | Oui (`INTERACTION_READ`) |

## Evenements Kafka

### Evenements produits

| Topic | Evenement | Description | Payload |
|---|---|---|---|
| `pharmacy-events` | `medication-dispensed` | Emis lors de la dispensation d'un medicament | `dispensingId`, `prescriptionId`, `patientId`, `medicationId`, `quantity`, `timestamp` |
| `pharmacy-events` | `stock-low-alert` | Emis lorsqu'un article atteint le seuil de reapprovisionnement | `stockItemId`, `medicationId`, `currentQuantity`, `reorderThreshold`, `timestamp` |
| `pharmacy-events` | `prescription-validated` | Emis lorsqu'une prescription est validee par le pharmacien | `prescriptionId`, `patientId`, `validatedBy`, `interactions`, `timestamp` |

### Evenements consommes

| Topic | Evenement | Description |
|---|---|---|
| `clinical-events` | `prescription-ordered` | Reception des prescriptions de medicaments emises par le service Clinical |

## Feign Clients

| Service cible | Usage |
|---|---|
| **Clinical** (`clinical-service`) | Recuperation des details des prescriptions, mise a jour du statut de dispensation |

## Configuration

```yaml
server:
  port: 8085

spring:
  datasource:
    url: jdbc:postgresql://localhost:5436/rofecare_pharmacy
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
      group-id: pharmacy-service
      auto-offset-reset: earliest

feign:
  client:
    config:
      clinical-service:
        url: http://localhost:8083
        connect-timeout: 5000
        read-timeout: 5000

jwt:
  secret: ${JWT_SECRET}

pharmacy:
  stock:
    low-stock-check-cron: "0 0 6 * * *"  # Verification quotidienne a 6h
    expiry-alert-days: 30                 # Alerte 30 jours avant peremption
```

## Schema de base de donnees (tables principales)

| Table | Description |
|---|---|
| `medications` | Catalogue des medicaments |
| `prescriptions` | Prescriptions recues du service Clinical |
| `dispensings` | Enregistrements de dispensation |
| `stock_items` | Articles en stock (par lot) |
| `drug_interactions` | Reference des interactions medicamenteuses |
| `pharmacy_orders` | Commandes fournisseurs |
| `order_items` | Lignes de commande |
| `suppliers` | Repertoire des fournisseurs |
| `controlled_substance_logs` | Journal des substances controlees |

## Tests

Le service Pharmacy dispose de **111 tests unitaires**. Les tests couvrent :

- **JUnit 5** pour les tests unitaires
- **Mockito** pour le mocking des dependances
- **Testcontainers** pour les tests d'integration (PostgreSQL)
- Validation du workflow de dispensation (prescription -> validation -> preparation -> dispensation)
- Verification des regles de detection d'interactions medicamenteuses
- Tests de la logique de gestion des stocks et des alertes de reapprovisionnement
- Couverture de la traçabilite des substances controlees
