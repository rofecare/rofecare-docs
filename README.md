# Rofecare HIS

> Systeme d'Information Hospitalier complet, modulaire et deployable en mode cloud, monolithe ou desktop hors-ligne.

[![Build](https://img.shields.io/badge/build-passing-brightgreen)]()
[![Tests](https://img.shields.io/badge/tests-1%20578%20passing-brightgreen)]()
[![Java](https://img.shields.io/badge/Java-21-blue)]()
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.1-green)]()
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)]()

---

## Presentation

Rofecare est un **Hospital Information System (HIS)** concu pour les etablissements de sante en Afrique et dans les pays en developpement. Il couvre l'ensemble du parcours patient : de l'identification a la facturation, en passant par les consultations, la pharmacie, le laboratoire et l'imagerie.

Le systeme se distingue par sa **double strategie de deploiement** : il fonctionne en tant qu'ensemble de microservices dans le cloud, ou en tant que monolithe unique pour un hopital local, avec une synchronisation bidirectionnelle entre les deux modes. Une application desktop Tauri permet un fonctionnement **100% hors-ligne**.

---

## Fonctionnalites cles

- **8 microservices metier** couvrant l'integralite du parcours de soins
- **Deploiement dual** : microservices (cloud) ou monolithe (hopital local)
- **Application desktop** Tauri avec fonctionnement hors-ligne complet
- **Synchronisation bidirectionnelle** Local <-> Cloud avec resolution de conflits
- **Interoperabilite** HL7 FHIR, DICOM, webhooks
- **Architecture hexagonale** et Domain-Driven Design
- **1 578 tests unitaires** avec couverture complete
- **Monitoring** : Prometheus, Grafana, ELK Stack, Zipkin

---

## Architecture

```
                              +---------------------+
                              |    API Gateway       |
                              |     (port 8080)      |
                              +----------+----------+
                                         |
            +----------------------------+----------------------------+
            |            |           |           |           |        |
       +----+----+ +----+----+ +----+----+ +----+----+ +----+----+   |
       |Identity | |Patient  | |Clinical | |MedTech  | |Pharmacy |   |
       |  8081   | |  8082   | |  8083   | |  8084   | |  8085   |   |
       +----+----+ +----+----+ +----+----+ +----+----+ +----+----+   |
            |            |           |           |           |        |
       +----+----+ +----+----+ +----+----+                           |
       |Finance  | |Platform | |Interop  |                           |
       |  8086   | |  8087   | |  8088   |                           |
       +----+----+ +----+----+ +----+----+                           |
            |            |           |                                |
    +-------+------------+-----------+--------------------------------+
    |                    Apache Kafka (9092)                          |
    +----------------------------------------------------------------+
    |              PostgreSQL x8  |  Redis  |  Monitoring             |
    +----------------------------------------------------------------+
```

Pour une vue detaillee, consultez la [documentation d'architecture](architecture/overview.md).

---

## Stack technique

| Couche            | Technologies                                      |
|-------------------|---------------------------------------------------|
| Backend           | Java 21, Spring Boot 3.4.1, Spring Cloud 2024.0.0 |
| Base de donnees   | PostgreSQL 16 (8 instances), Redis 7               |
| Migrations        | Flyway                                             |
| Messaging         | Apache Kafka (Confluent 7.6.0)                     |
| API Gateway       | Spring Cloud Gateway                               |
| Service Discovery | Eureka                                             |
| Configuration     | Spring Cloud Config Server                         |
| Resilience        | Resilience4j (Circuit Breaker, Retry, Rate Limiter)|
| Tests             | JUnit 5, Mockito, Testcontainers, AssertJ, ArchUnit|
| Build             | Maven 3.9+                                         |
| Frontend          | Nuxt 3, Vue 3                                      |
| Desktop           | Tauri 2.x (Rust, WebView2)                         |
| CI/CD             | GitHub Actions                                     |
| Monitoring        | Prometheus, Grafana, ELK Stack, Zipkin              |

---

## Demarrage rapide

### Prerequis

- Java 21 (JDK)
- Maven 3.9+
- Docker & Docker Compose
- Node.js 20+ (pour le frontend)
- Rust (pour l'application desktop Tauri)

### Lancement en mode microservices

```bash
# Demarrer l'infrastructure
docker compose up -d postgres redis kafka zookeeper

# Demarrer les services d'infrastructure
./mvnw -pl rofecare-config-server spring-boot:run &
./mvnw -pl rofecare-discovery-server spring-boot:run &
./mvnw -pl rofecare-api-gateway spring-boot:run &

# Demarrer les services metier
./mvnw -pl rofecare-identity-service spring-boot:run &
./mvnw -pl rofecare-patient-service spring-boot:run &
./mvnw -pl rofecare-clinical-service spring-boot:run &
# ... autres services
```

### Lancement en mode monolithe

```bash
docker compose up -d postgres redis

./mvnw -pl rofecare-server spring-boot:run -Dspring.profiles.active=monolith
```

Pour des instructions detaillees, consultez le [guide de demarrage](getting-started.md).

---

## Structure du projet

```
rofecare/
├── rofecare-identity-service/       # Authentification, utilisateurs, roles, JWT
├── rofecare-patient-service/        # Dossiers patients, triage, admissions
├── rofecare-clinical-service/       # Consultations, diagnostics, prescriptions
├── rofecare-medtech-service/        # Laboratoire, imagerie, equipements
├── rofecare-pharmacy-service/       # Medicaments, dispensation, stock
├── rofecare-finance-service/        # Facturation, paiements, assurances
├── rofecare-platform-service/       # Notifications, audit, documents, planification
├── rofecare-interoperability-service/ # HL7/FHIR, DICOM, webhooks
├── rofecare-api-gateway/            # Routage, CORS, rate limiting
├── rofecare-discovery-server/       # Eureka service registry
├── rofecare-config-server/          # Configuration centralisee
├── rofecare-server/                 # Aggregateur monolithe
├── rofecare-common/                 # Bibliotheque partagee
├── rofecare-desktop/                # Application Tauri
├── rofecare-docs/                   # Documentation du projet
├── docker-compose.yml               # Orchestration des services
└── pom.xml                          # POM parent Maven
```

---

## Documentation

| Section                                                    | Description                                      |
|------------------------------------------------------------|--------------------------------------------------|
| [Vue d'ensemble architecture](architecture/overview.md)    | Diagrammes, patterns, decisions techniques        |
| [Sous-domaines DDD](architecture/subdomains.md)            | Bounded contexts, aggregats, evenements           |
| [Deploiement dual](architecture/dual-deployment.md)        | Cloud vs monolithe vs desktop, synchronisation    |
| [Services metier](services/)                               | Documentation detaillee de chaque microservice    |
| [Infrastructure](infrastructure/)                          | PostgreSQL, Redis, Kafka, monitoring              |
| [Deploiement](deployment/)                                 | Docker, CI/CD, environnements                     |
| [Developpement](development/)                              | Conventions, contribution, tests                  |

---

## Contribuer

Les contributions sont les bienvenues. Veuillez consulter le [guide de contribution](development/contributing.md) pour les conventions de code, la strategie Git et le processus de revue.

### Strategie Git

```
main --> develop --> feature/*, bugfix/*, hotfix/*, release/*
```

### Conventional Commits

```
feat|fix|docs|style|refactor|test|chore|perf|ci(scope): description
```

---

## Licence

Ce projet est distribue sous licence **Apache 2.0**. Voir le fichier [LICENSE](../LICENSE) pour plus de details.

---

## Contact

Pour toute question ou suggestion, veuillez ouvrir une issue sur le depot GitHub du projet.
