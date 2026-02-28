# Guide de demarrage pour les developpeurs

Guide complet pour configurer l'environnement de developpement Rofecare HIS et commencer a contribuer au projet.

---

## Table des matieres

- [Prerequisites](#prerequisites)
- [Cloner le repository](#cloner-le-repository)
- [Ordre de compilation](#ordre-de-compilation)
- [Demarrer l'infrastructure locale](#demarrer-linfrastructure-locale)
- [Executer un service dans l'IDE](#executer-un-service-dans-lide)
- [Executer tous les services via Docker](#executer-tous-les-services-via-docker)
- [Configuration de l'IDE](#configuration-de-lide)
- [Structure du projet](#structure-du-projet)
- [Architecture hexagonale](#architecture-hexagonale)
- [Documentation API](#documentation-api)
- [Tests](#tests)

---

## Prerequisites

### Logiciels requis

| Outil          | Version        | Lien de telechargement                                    |
| -------------- | -------------- | --------------------------------------------------------- |
| Java (JDK)     | 21 (Temurin)   | https://adoptium.net/                                     |
| Maven          | 3.9+           | https://maven.apache.org/download.cgi                     |
| Docker Desktop | Latest         | https://www.docker.com/products/docker-desktop            |
| Git            | 2.x            | https://git-scm.com/                                      |
| IDE            | IntelliJ IDEA (recommande) | https://www.jetbrains.com/idea/                |

### Verification de l'environnement

```bash
# Java
java -version
# openjdk version "21.x.x" (Eclipse Adoptium)

# Maven
mvn -version
# Apache Maven 3.9.x

# Docker
docker --version
docker compose version

# Git
git --version
```

### Configuration Java

Assurez-vous que `JAVA_HOME` pointe vers le JDK 21 :

```bash
# macOS / Linux
export JAVA_HOME=$(/usr/libexec/java_home -v 21)  # macOS
export JAVA_HOME=/usr/lib/jvm/temurin-21-jdk       # Linux

# Windows (PowerShell)
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-21"
```

---

## Cloner le repository

```bash
# Cloner le backend
git clone https://github.com/your-org/rofecare-server.git
cd rofecare-server

# Se placer sur la branche develop
git checkout develop

# Cloner le frontend (optionnel)
git clone https://github.com/your-org/rofecare-frontend.git
```

---

## Ordre de compilation

L'ordre de compilation est important car les services dependent du module commun.

### Etape 1 : Compiler et installer `rofecare-common`

```bash
cd rofecare-common
mvn clean install
```

Ce module contient :

- Les DTOs partages entre services
- Les interfaces communes
- Les utilitaires et annotations partagees
- Les events Kafka partages

> **Important** : cette etape installe le JAR dans le repository Maven local (`~/.m2/repository`). Elle doit etre repetee a chaque modification de `rofecare-common`.

### Etape 2 : Compiler un service individuel

```bash
cd rofecare-patient-service
mvn clean package
```

### Etape 3 : Compiler tous les services

```bash
# Depuis la racine du projet
cd rofecare-server
mvn clean package -DskipTests
```

---

## Demarrer l'infrastructure locale

Pour le developpement, demarrez uniquement l'infrastructure (bases de donnees, Redis, Kafka) via Docker, et executez les services Java dans votre IDE.

```bash
# Demarrer uniquement l'infrastructure
docker compose up -d \
  identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka kafka-ui
```

### Verifier l'infrastructure

```bash
# Etat des conteneurs
docker compose ps

# Tester PostgreSQL
docker compose exec identity-db psql -U identity_user -d identity_db -c "SELECT 1;"

# Tester Redis
docker compose exec redis redis-cli -a password ping

# Interface Kafka UI
# Ouvrir http://localhost:9000
```

### Arreter l'infrastructure

```bash
# Arreter sans supprimer les donnees
docker compose stop identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka kafka-ui

# Ou tout arreter
docker compose down

# Arreter et reinitialiser les donnees
docker compose down -v
```

---

## Executer un service dans l'IDE

### Configuration du profil Spring

Chaque service peut etre demarre avec le profil `local` pour utiliser l'infrastructure Docker locale :

```
# VM Options ou Run Configuration
-Dspring.profiles.active=local
```

### Variables d'environnement pour le profil local

```properties
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5434/patient_db
SPRING_DATASOURCE_USERNAME=patient_user
SPRING_DATASOURCE_PASSWORD=password
SPRING_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
SPRING_DATA_REDIS_HOST=localhost
SPRING_DATA_REDIS_PASSWORD=password
EUREKA_CLIENT_ENABLED=false
```

> **Astuce** : desactivez Eureka (`EUREKA_CLIENT_ENABLED=false`) pour lancer un service individuellement sans le Discovery Server.

### Ports par service

| Service                    | Port  | Classe principale                    |
| -------------------------- | ----- | ------------------------------------ |
| Identity Service           | 8081  | `IdentityServiceApplication`        |
| Patient Service            | 8082  | `PatientServiceApplication`         |
| Clinical Service           | 8083  | `ClinicalServiceApplication`        |
| Medical Technology Service | 8084  | `MedicalTechnologyServiceApplication` |
| Pharmacy Service           | 8085  | `PharmacyServiceApplication`        |
| Finance Service            | 8086  | `FinanceServiceApplication`         |
| Platform Service           | 8087  | `PlatformServiceApplication`        |
| Interoperability Service   | 8088  | `InteroperabilityServiceApplication`|

---

## Executer tous les services via Docker

Pour tester l'ensemble de la plateforme :

```bash
# Compiler tous les services
mvn clean package -DskipTests

# Construire les images Docker
docker compose build

# Demarrer l'infrastructure
docker compose up -d identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka

# Attendre 30 secondes

# Demarrer Spring Cloud
docker compose up -d config-server
# Attendre 30 secondes
docker compose up -d discovery-server
# Attendre 30 secondes
docker compose up -d gateway

# Demarrer les services metier
docker compose up -d identity-service patient-service clinical-service \
  medical-technology-service pharmacy-service finance-service \
  platform-service interoperability-service
```

Ou utilisez le script :

```bash
./start-all.sh
```

---

## Configuration de l'IDE

### IntelliJ IDEA (recommande)

#### Import du projet

1. **File > Open** et selectionnez le repertoire `rofecare-server`
2. IntelliJ detectera automatiquement le projet Maven
3. Attendez l'indexation et le telechargement des dependances

#### Plugins recommandes

| Plugin                    | Usage                                |
| ------------------------- | ------------------------------------ |
| Lombok                    | Support des annotations Lombok       |
| MapStruct Support         | Navigation et completion MapStruct   |
| Docker                    | Gestion des conteneurs               |
| Spring Boot               | Inclus dans IntelliJ Ultimate        |
| Database Tools            | Inclus dans IntelliJ Ultimate        |

#### Configuration Lombok

1. **File > Settings > Build > Compiler > Annotation Processors**
2. Cocher **Enable annotation processing**

#### Run Configuration recommandee

Pour chaque service, creez une Run Configuration :

```
Type: Spring Boot
Main class: com.rofecare.patient.PatientServiceApplication
Active profiles: local
VM options: -Xms256m -Xmx512m
Environment variables: (voir section Variables d'environnement)
```

### VS Code

#### Extensions recommandees

| Extension                          | Usage                           |
| ---------------------------------- | ------------------------------- |
| Extension Pack for Java            | Support Java complet            |
| Spring Boot Extension Pack         | Support Spring Boot             |
| Docker                             | Gestion des conteneurs          |
| Lombok Annotations Support         | Support Lombok                  |

#### Fichier `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "Patient Service",
      "request": "launch",
      "mainClass": "com.rofecare.patient.PatientServiceApplication",
      "vmArgs": "-Dspring.profiles.active=local -Xms256m -Xmx512m",
      "env": {
        "SPRING_DATASOURCE_URL": "jdbc:postgresql://localhost:5434/patient_db",
        "SPRING_DATASOURCE_USERNAME": "patient_user",
        "SPRING_DATASOURCE_PASSWORD": "password",
        "EUREKA_CLIENT_ENABLED": "false"
      }
    }
  ]
}
```

---

## Structure du projet

```
rofecare-server/
├── rofecare-common/                    # Module partage
│   ├── src/main/java/com/rofecare/common/
│   │   ├── dto/                        # DTOs partages
│   │   ├── event/                      # Events Kafka
│   │   ├── exception/                  # Exceptions communes
│   │   └── util/                       # Utilitaires
│   └── pom.xml
│
├── rofecare-identity-service/          # Service d'identite et authentification
├── rofecare-patient-service/           # Service de gestion des patients
├── rofecare-clinical-service/          # Service clinique
├── rofecare-medical-technology-service/ # Service de technologie medicale
├── rofecare-pharmacy-service/          # Service de pharmacie
├── rofecare-finance-service/           # Service de finance
├── rofecare-platform-service/          # Service de plateforme
├── rofecare-interoperability-service/  # Service d'interoperabilite
│
├── rofecare-config-server/             # Spring Cloud Config Server
├── rofecare-discovery-server/          # Eureka Discovery Server
├── rofecare-gateway/                   # Spring Cloud Gateway
├── rofecare-server/                    # Monolithe (tous modules)
│
├── docker-compose.yml                  # Docker Compose principal
├── docker-compose.prod.yml             # Overlay production
├── docker-compose.monitoring.yml       # Stack monitoring
├── .env.example                        # Variables d'environnement
├── start-all.sh                        # Script de demarrage complet
├── start-infra.sh                      # Script infrastructure
└── pom.xml                             # POM parent
```

### Structure d'un service (exemple : Patient Service)

```
rofecare-patient-service/
├── src/
│   ├── main/
│   │   ├── java/com/rofecare/patient/
│   │   │   ├── PatientServiceApplication.java
│   │   │   ├── domain/                 # Couche domaine
│   │   │   │   ├── model/             # Entites de domaine
│   │   │   │   ├── port/              # Ports (interfaces)
│   │   │   │   │   ├── in/            # Ports entrants (use cases)
│   │   │   │   │   └── out/           # Ports sortants (persistence)
│   │   │   │   └── service/           # Services de domaine
│   │   │   ├── application/            # Couche application
│   │   │   │   ├── usecase/           # Implementation des use cases
│   │   │   │   └── mapper/           # MapStruct mappers
│   │   │   └── infrastructure/         # Couche infrastructure
│   │   │       ├── adapter/
│   │   │       │   ├── in/            # Adaptateurs entrants
│   │   │       │   │   ├── rest/      # Controllers REST
│   │   │       │   │   └── kafka/     # Consumers Kafka
│   │   │       │   └── out/           # Adaptateurs sortants
│   │   │       │       ├── persistence/ # Repositories JPA
│   │   │       │       └── kafka/     # Producers Kafka
│   │   │       └── config/            # Configuration Spring
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-local.yml
│   │       └── application-docker.yml
│   └── test/
│       └── java/com/rofecare/patient/
│           ├── domain/                 # Tests unitaires domaine
│           ├── application/            # Tests use cases
│           ├── infrastructure/         # Tests d'integration
│           └── architecture/           # Tests ArchUnit
├── Dockerfile
└── pom.xml
```

---

## Architecture hexagonale

Le projet suit l'architecture hexagonale (ports & adapters). Comprendre cette architecture est essentiel pour contribuer correctement.

### Principes fondamentaux

```
                    +---------------------+
  Controllers  ---->|  Port IN (interface) |
  Kafka Consumer -->|                     |
                    |     DOMAIN          |
                    |   (pur Java,        |
                    |    pas de Spring)   |
                    |                     |
                    |  Port OUT (interface)|----> JPA Repository
                    |                     |----> Kafka Producer
                    +---------------------+
```

### Regles architecturales

| Regle                                              | Description                                              |
| -------------------------------------------------- | -------------------------------------------------------- |
| Le domaine est independant                         | Aucune dependance Spring, JPA ou framework dans `domain/` |
| Communication via ports                            | Le domaine communique uniquement via des interfaces       |
| Les adaptateurs implementent les ports             | Les classes d'infrastructure implementent les interfaces  |
| Flux entrant : Controller -> Use Case -> Domain    | Les requetes traversent les couches de l'exterieur vers l'interieur |
| Flux sortant : Domain -> Port OUT -> Adapter       | La persistance est abstraite derriere une interface       |

### Exemple concret

```java
// Port IN (domain/port/in/)
public interface RegisterPatientUseCase {
    PatientId register(RegisterPatientCommand command);
}

// Port OUT (domain/port/out/)
public interface PatientRepository {
    Patient save(Patient patient);
    Optional<Patient> findById(PatientId id);
}

// Use Case (application/usecase/)
@RequiredArgsConstructor
public class RegisterPatientUseCaseImpl implements RegisterPatientUseCase {
    private final PatientRepository patientRepository; // Port OUT

    @Override
    public PatientId register(RegisterPatientCommand command) {
        Patient patient = Patient.create(command);
        return patientRepository.save(patient).getId();
    }
}

// Adapter IN (infrastructure/adapter/in/rest/)
@RestController
@RequiredArgsConstructor
public class PatientController {
    private final RegisterPatientUseCase registerPatient; // Port IN

    @PostMapping("/api/v1/patients")
    public ResponseEntity<PatientResponse> register(@RequestBody RegisterPatientRequest request) {
        PatientId id = registerPatient.register(toCommand(request));
        return ResponseEntity.created(...).build();
    }
}

// Adapter OUT (infrastructure/adapter/out/persistence/)
@Repository
@RequiredArgsConstructor
public class JpaPatientRepository implements PatientRepository {
    private final SpringDataPatientRepository springRepo;
    private final PatientPersistenceMapper mapper;

    @Override
    public Patient save(Patient patient) {
        PatientEntity entity = mapper.toEntity(patient);
        return mapper.toDomain(springRepo.save(entity));
    }
}
```

### ArchUnit : verification automatique

Les regles architecturales sont verifiees automatiquement par des tests ArchUnit :

```java
@AnalyzeClasses(packages = "com.rofecare.patient")
class ArchitectureTest {

    @ArchTest
    static final ArchRule domain_should_not_depend_on_spring =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("org.springframework..");

    @ArchTest
    static final ArchRule domain_should_not_depend_on_infrastructure =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");
}
```

---

## Documentation API

Chaque service expose sa documentation API via SpringDoc OpenAPI (Swagger).

### Acces a la documentation

| Service                    | URL Swagger UI                                    |
| -------------------------- | ------------------------------------------------- |
| Identity Service           | http://localhost:8081/swagger-ui.html              |
| Patient Service            | http://localhost:8082/swagger-ui.html              |
| Clinical Service           | http://localhost:8083/swagger-ui.html              |
| Medical Technology Service | http://localhost:8084/swagger-ui.html              |
| Pharmacy Service           | http://localhost:8085/swagger-ui.html              |
| Finance Service            | http://localhost:8086/swagger-ui.html              |
| Platform Service           | http://localhost:8087/swagger-ui.html              |
| Interoperability Service   | http://localhost:8088/swagger-ui.html              |

### Specification OpenAPI (JSON)

```bash
# Obtenir la specification OpenAPI au format JSON
curl http://localhost:8082/v3/api-docs

# Obtenir la specification au format YAML
curl http://localhost:8082/v3/api-docs.yaml
```

### Via le Gateway

En mode microservices, accedez a la documentation via le Gateway :

```
http://localhost:8080/api/v1/patients/swagger-ui.html
```

---

## Tests

### Executer les tests unitaires

```bash
# Tests d'un service specifique
cd rofecare-patient-service
mvn test

# Tests de tous les services
cd rofecare-server
mvn test
```

### Executer les tests d'integration

Les tests d'integration utilisent **Testcontainers** pour demarrer automatiquement les bases de donnees necessaires :

```bash
# Tests d'integration (incluant les tests unitaires)
mvn verify

# Tests d'integration uniquement
mvn failsafe:integration-test failsafe:verify
```

> **Note** : Docker doit etre en cours d'execution pour les tests d'integration avec Testcontainers.

### Rapport de couverture

```bash
# Generer le rapport JaCoCo
mvn verify -Pjacoco

# Le rapport est genere dans target/site/jacoco/index.html
```

### Executer un test specifique

```bash
# Un test specifique
mvn test -Dtest=PatientServiceTest

# Une methode de test specifique
mvn test -Dtest=PatientServiceTest#shouldRegisterNewPatient

# Tests par pattern
mvn test -Dtest="*Integration*"
```
