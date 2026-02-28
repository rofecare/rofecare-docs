# Architecture - Vue d'ensemble

Ce document presente l'architecture globale du systeme Rofecare HIS, les choix techniques et les patterns utilises.

---

## Table des matieres

- [Diagramme de contexte systeme](#diagramme-de-contexte-systeme)
- [Diagramme de containers](#diagramme-de-containers)
- [Architecture hexagonale](#architecture-hexagonale)
- [Structure des packages](#structure-des-packages)
- [Patterns de communication](#patterns-de-communication)
- [Carte des ports](#carte-des-ports)
- [Decisions techniques](#decisions-techniques)

---

## Diagramme de contexte systeme

Le diagramme suivant represente le systeme Rofecare dans son environnement, avec les acteurs et systemes externes.

```
    +----------------+      +----------------+      +-----------------+
    |   Medecin      |      |  Pharmacien    |      |  Administrateur |
    +-------+--------+      +-------+--------+      +--------+--------+
            |                        |                        |
            +------------------------+------------------------+
                                     |
                            +--------+--------+
                            |                 |
                            |    Rofecare     |
                            |      HIS       |
                            |                 |
                            +--------+--------+
                                     |
            +------------------------+------------------------+
            |                        |                        |
    +-------+--------+      +-------+--------+      +--------+--------+
    | Systemes HL7   |      | Systemes DICOM |      |  Assurances     |
    | FHIR externes  |      | (imagerie)     |      |  (tiers payant) |
    +----------------+      +----------------+      +-----------------+
```

**Acteurs principaux :**
- **Personnel medical** : medecins, infirmiers, specialistes
- **Pharmaciens** : gestion des medicaments et dispensation
- **Administrateurs** : gestion financiere, configuration systeme
- **Techniciens de laboratoire** : analyses, resultats
- **Personnel d'accueil** : enregistrement patients, triage

**Systemes externes :**
- Systemes HL7 FHIR pour l'interoperabilite avec d'autres etablissements
- Systemes DICOM pour l'imagerie medicale
- Organismes d'assurance pour le tiers payant

---

## Diagramme de containers

```
+------------------------------------------------------------------+
|                         API Gateway (8080)                        |
|              Spring Cloud Gateway - Routage, CORS,               |
|              Rate Limiting, Authentication Filter                |
+---+------+------+------+------+------+------+------+-------------+
    |      |      |      |      |      |      |      |
    v      v      v      v      v      v      v      v
+------+ +------+ +------+ +------+ +------+ +------+ +------+ +------+
|Ident.| |Pati. | |Clin. | |Med.  | |Phar. | |Fin.  | |Plat. | |Inter.|
|8081  | |8082  | |8083  | |Tech  | |8085  | |8086  | |8087  | |8088  |
|      | |      | |      | |8084  | |      | |      | |      | |      |
+--+---+ +--+---+ +--+---+ +--+---+ +--+---+ +--+---+ +--+---+ +--+---+
   |        |        |        |        |        |        |        |
   v        v        v        v        v        v        v        v
+------+ +------+ +------+ +------+ +------+ +------+ +------+ +------+
|PG    | |PG    | |PG    | |PG    | |PG    | |PG    | |PG    | |PG    |
|5432  | |5433  | |5434  | |5435  | |5436  | |5437  | |5438  | |5439  |
+------+ +------+ +------+ +------+ +------+ +------+ +------+ +------+

+------------------------------------------------------------------+
|                     Apache Kafka (9092)                           |
|          Bus d'evenements asynchrones inter-services             |
+------------------------------------------------------------------+

+------------------+  +-------------------+  +--------------------+
|   Redis (6379)   |  | Discovery (8761)  |  | Config Srv (8888)  |
|   Cache, Session |  | Eureka Registry   |  | Config centralisee |
+------------------+  +-------------------+  +--------------------+

+------------------+  +-------------------+  +--------------------+
| Prometheus(9090) |  | Grafana (3001)    |  | Zipkin (9411)      |
| Metriques        |  | Dashboards        |  | Distributed Tracing|
+------------------+  +-------------------+  +--------------------+

+------------------+  +-------------------+
| Elastic (9200)   |  | Kibana (5601)     |
| Logs centralises |  | Visualisation logs|
+------------------+  +-------------------+
```

---

## Architecture hexagonale

Chaque microservice metier suit une **architecture hexagonale** (Ports & Adapters), garantissant une separation stricte entre la logique metier et les details d'infrastructure.

### Principes

1. **Le domaine est au centre** : aucune dependance vers l'exterieur
2. **Les ports definissent les contrats** : interfaces d'entree (use cases) et de sortie (repositories)
3. **Les adapters implementent les ports** : REST controllers, JPA repositories, Feign clients
4. **Inversion de dependance** : l'infrastructure depend du domaine, jamais l'inverse

### Schema de l'architecture hexagonale

```
                    +---------------------------+
                    |    Adapter Input (REST)    |
                    |    Controllers             |
                    +------------+--------------+
                                 |
                                 v
                    +------------+--------------+
                    |     Port Input             |
                    |     (Use Case interfaces)  |
                    +------------+--------------+
                                 |
                                 v
          +----------------------------------------------+
          |                                              |
          |              APPLICATION LAYER               |
          |                                              |
          |   +--------------------------------------+   |
          |   |          Use Case Impl               |   |
          |   |   (orchestration, validation,        |   |
          |   |    mapping, transaction)             |   |
          |   +--------------------------------------+   |
          |                                              |
          +----------------------------------------------+
                                 |
                                 v
          +----------------------------------------------+
          |                                              |
          |               DOMAIN LAYER                   |
          |                                              |
          |   Models, Value Objects, Aggregates          |
          |   Domain Services, Domain Events             |
          |   Repository Interfaces (Output Ports)       |
          |   Business Exceptions                        |
          |                                              |
          +----------------------------------------------+
                                 ^
                                 |
                    +------------+--------------+
                    |     Port Output            |
                    |  (Repository interfaces)   |
                    +------------+--------------+
                                 ^
                                 |
          +----------------------+------------------------+
          |                      |                        |
+---------+--------+  +---------+--------+  +------------+-------+
| Adapter Output   |  | Adapter Output   |  | Adapter Output     |
| JPA Persistence  |  | Feign Clients    |  | Kafka Publishers   |
+------------------+  +------------------+  +--------------------+
```

### Flux d'une requete

1. Le **REST Controller** (adapter input) recoit la requete HTTP
2. Il la convertit en **Request DTO** et appelle le **Use Case** (port input)
3. Le **Use Case Implementation** orchestre la logique applicative
4. Il utilise les **modeles de domaine** et les **domain services**
5. Les donnees sont persistees via les **Repository interfaces** (port output)
6. Les **JPA Adapters** (adapter output) implementent la persistance reelle
7. Les evenements de domaine sont publies via **Kafka**
8. Le **Response DTO** est retourne au controller

---

## Structure des packages

Chaque service metier suit la meme structure de packages, alignee sur l'architecture hexagonale.

```
com.rofecare.{service}/
│
├── domain/
│   ├── model/              # Entites de domaine, Value Objects, Aggregates
│   ├── repository/         # Interfaces de repository (output ports)
│   ├── service/            # Services de domaine (logique metier pure)
│   ├── event/              # Evenements de domaine
│   └── exception/          # Exceptions metier
│
├── application/
│   ├── port/
│   │   ├── input/          # Interfaces de use cases (input ports)
│   │   └── output/         # Interfaces de repository (output ports)
│   ├── usecase/            # Implementations des use cases
│   ├── dto/
│   │   ├── request/        # DTOs de requete
│   │   └── response/       # DTOs de reponse
│   ├── mapper/             # Mappers MapStruct (Entity <-> DTO)
│   └── listener/           # Kafka event listeners
│
└── infrastructure/
    ├── adapter/
    │   ├── input/
    │   │   └── rest/       # REST Controllers (Spring MVC)
    │   └── output/
    │       ├── persistence/# Entites JPA, Spring Data repositories
    │       └── client/     # Feign clients (appels inter-services)
    ├── config/             # Configuration Spring (beans, security, etc.)
    └── security/           # Configuration securite (JWT, filtres)
```

### Conventions

- **Modeles de domaine** : POJOs purs, sans annotations JPA ni Spring
- **Entites JPA** : suffixees par `Entity`, dans le package persistence
- **Mappers** : interfaces MapStruct pour la conversion entre couches
- **DTOs** : suffixes `Request` et `Response`
- **Use Cases** : prefixes par le verbe d'action (`CreatePatient`, `FindConsultation`)

---

## Patterns de communication

### Communication synchrone

Les appels inter-services synchrones utilisent **OpenFeign** avec **Resilience4j** pour la resilience.

```
Service A  ----HTTP/REST----> Service B
              (Feign Client)

              + Circuit Breaker (Resilience4j)
              + Retry (3 tentatives, backoff exponentiel)
              + Rate Limiter
              + Timeout (3s connect, 10s read)
```

**Cas d'usage** : lecture de donnees en temps reel, validation croisee, enrichissement de reponse.

**Exemples :**
- Clinical -> Patient : recuperer les informations patient lors d'une consultation
- Finance -> Patient : verifier le statut d'assurance pour la facturation
- Pharmacy -> Clinical : valider une prescription avant dispensation

### Communication asynchrone

Les evenements inter-services utilisent **Apache Kafka** pour le decouplage et la fiabilite.

```
Service A  ----Kafka Topic----> Service B
              (Domain Event)    Service C
                                Service D
```

**Convention de nommage des topics** : `rofecare.{service}.{event-name}`

**Evenements principaux :**

| Evenement                     | Producteur  | Consommateurs              |
|-------------------------------|-------------|----------------------------|
| `patient-registered`          | Patient     | Clinical, Finance, Platform|
| `consultation-completed`      | Clinical    | Finance, Platform          |
| `prescription-ordered`        | Clinical    | Pharmacy                   |
| `medication-dispensed`        | Pharmacy    | Finance, Clinical          |
| `laboratory-result-available` | MedTech     | Clinical, Platform         |
| `invoice-created`             | Finance     | Platform                   |
| `user-created`                | Identity    | Platform                   |

**Garanties :**
- **At-least-once delivery** : les consommateurs sont idempotents
- **Ordered processing** : partitionnement par ID d'entite
- **Dead Letter Queue** : les messages en echec sont rediriges

---

## Carte des ports

### Services metier

| Service              | Port App | Port DB | Base de donnees       |
|----------------------|----------|---------|-----------------------|
| Identity Service     | 8081     | 5432    | rofecare_identity     |
| Patient Service      | 8082     | 5433    | rofecare_patient      |
| Clinical Service     | 8083     | 5434    | rofecare_clinical     |
| MedTech Service      | 8084     | 5435    | rofecare_medtech      |
| Pharmacy Service     | 8085     | 5436    | rofecare_pharmacy     |
| Finance Service      | 8086     | 5437    | rofecare_finance      |
| Platform Service     | 8087     | 5438    | rofecare_platform     |
| Interop Service      | 8088     | 5439    | rofecare_interop      |

### Services d'infrastructure

| Service              | Port     | Role                          |
|----------------------|----------|-------------------------------|
| API Gateway          | 8080     | Point d'entree unique, routage|
| Discovery Server     | 8761     | Registre de services Eureka   |
| Config Server        | 8888     | Configuration centralisee     |

### Infrastructure externe

| Composant            | Port     | Role                          |
|----------------------|----------|-------------------------------|
| Redis                | 6379     | Cache, sessions               |
| Kafka                | 9092     | Bus d'evenements              |
| Zookeeper            | 2181     | Coordination Kafka            |
| Prometheus           | 9090     | Collecte de metriques         |
| Grafana              | 3001     | Dashboards de monitoring      |
| Elasticsearch        | 9200     | Stockage de logs              |
| Kibana               | 5601     | Visualisation de logs         |
| Zipkin               | 9411     | Tracing distribue             |
| Kafka UI             | 8180     | Interface Kafka               |

---

## Decisions techniques

### Java 21

Java 21 est la version LTS la plus recente, offrant :
- **Virtual Threads** (Project Loom) pour une meilleure scalabilite I/O
- **Pattern Matching** et **Records** pour un code plus expressif
- **Sealed Classes** pour une modelisation de domaine plus sure
- Support LTS garanti jusqu'en 2029

### Spring Boot 3.4.1 + Spring Cloud 2024.0.0

- Ecosysteme complet pour les microservices
- Integration native avec les outils d'observabilite (Micrometer, Actuator)
- Support GraalVM Native Image pour des temps de demarrage reduits
- Communaute large et documentation riche

### PostgreSQL 16

- Base de donnees relationnelle robuste et performante
- Support natif JSON/JSONB pour les donnees semi-structurees
- Fonctionnalites avancees : partitionnement, full-text search, CTE recursives
- **Database-per-service** : chaque microservice possede sa propre instance, garantissant l'isolation des donnees et l'autonomie de deploiement

### Apache Kafka

- Bus d'evenements distribue hautement disponible
- Garanties de livraison configurables (at-least-once, exactly-once)
- Retention des evenements pour le replay
- Scalabilite horizontale via le partitionnement

### Architecture hexagonale

- Separation stricte domaine / infrastructure
- Testabilite maximale de la logique metier (tests unitaires sans infrastructure)
- Interchangeabilite des adapters (ex: changer de base de donnees sans toucher au domaine)
- Alignement avec les principes DDD

### Resilience4j

Choisi plutot que Hystrix (deprecie) pour :
- Design modulaire et leger
- Integration native avec Spring Boot 3
- Supporte : Circuit Breaker, Retry, Rate Limiter, Bulkhead, TimeLimiter

### Flyway

- Migrations de schema versionnees et reproductibles
- Historique complet des changements de base de donnees
- Compatible avec le deploiement automatise (CI/CD)

---

*Voir aussi :*
- [Sous-domaines et Bounded Contexts](subdomains.md)
- [Strategie de deploiement dual](dual-deployment.md)
- [Documentation des services](../services/)
