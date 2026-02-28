# Strategie de deploiement dual

Rofecare propose une architecture unique permettant de deployer le meme code source en trois modes distincts : **microservices** (cloud), **monolithe** (hopital local) et **desktop** (poste autonome). Ce document decrit chaque mode, leurs differences, le mecanisme de synchronisation et les criteres de choix.

---

## Table des matieres

- [Vue d'ensemble](#vue-densemble)
- [Mode Microservices (Cloud)](#mode-microservices-cloud)
- [Mode Monolithe (Hopital local)](#mode-monolithe-hopital-local)
- [Mode Desktop (Tauri)](#mode-desktop-tauri)
- [Tableau comparatif](#tableau-comparatif)
- [Mecanisme de synchronisation](#mecanisme-de-synchronisation)
- [Matrice de decision](#matrice-de-decision)
- [Configuration par mode](#configuration-par-mode)

---

## Vue d'ensemble

```
+-----------------------------------------------------------------------+
|                    ROFECARE - Modes de deploiement                     |
+-----------------------------------------------------------------------+
|                                                                       |
|   +-------------------+    +------------------+    +----------------+ |
|   |   MICROSERVICES   |    |    MONOLITHE     |    |    DESKTOP     | |
|   |     (Cloud)       |    |  (Hopital local) |    |    (Tauri)     | |
|   +-------------------+    +------------------+    +----------------+ |
|   | 8 containers      |    | 1 JAR            |    | 1 executable   | |
|   | 8 bases PostgreSQL |    | 1 base PostgreSQL|    | PostgreSQL     | |
|   | Kafka             |    | Events In-Memory |    |   embarque     | |
|   | Eureka            |    | Discovery off    |    | JRE 21 sidecar | |
|   | Config Server     |    | Config locale    |    | Nuxt SSG       | |
|   +-------------------+    +------------------+    +----------------+ |
|            |                        |                       |         |
|            +------------------------+-----------------------+         |
|                                     |                                 |
|                          +----------+----------+                      |
|                          |  Synchronisation    |                      |
|                          |  bidirectionnelle   |                      |
|                          +---------------------+                      |
+-----------------------------------------------------------------------+
```

La cle de cette architecture repose sur l'**architecture hexagonale** : la logique metier (domaine et application) est identique dans tous les modes. Seuls les **adapters d'infrastructure** changent selon le mode de deploiement.

---

## Mode Microservices (Cloud)

Le mode microservices est concu pour les deploiements en cloud, les grands hopitaux ou les reseaux d'etablissements necessitant une haute disponibilite et une scalabilite horizontale.

### Architecture

```
                           Load Balancer
                                |
                        +-------+-------+
                        |  API Gateway  |
                        |    (8080)     |
                        +-------+-------+
                                |
       +----------+----------+--+--+----------+----------+
       |          |          |     |          |          |
   +---+---+ +---+---+ +---+---+ +---+---+ +---+---+ +---+---+
   |Ident. | |Patient| |Clinic.| |MedTech| |Pharma.| |Finance|
   | x2    | | x2    | | x3    | | x2    | | x2    | | x2    |
   +---+---+ +---+---+ +---+---+ +---+---+ +---+---+ +---+---+
       |          |          |         |         |         |
   +---+---+ +---+---+ +---+---+ +---+---+ +---+---+ +---+---+
   |PG     | |PG     | |PG     | |PG     | |PG     | |PG     |
   |replica| |replica| |replica| |replica| |replica| |replica|
   +-------+ +-------+ +-------+ +-------+ +-------+ +-------+

   +---+---+ +---+---+
   |Platf. | |Interop|
   | x2    | | x1    |
   +---+---+ +---+---+
       |          |
   +---+---+ +---+---+
   |PG     | |PG     |
   |replica| |replica|
   +-------+ +-------+

   +----------------------------------------------------------+
   |              Apache Kafka Cluster (3 brokers)             |
   +----------------------------------------------------------+
   +------------------+ +------------------+ +----------------+
   | Redis Cluster    | | Eureka (x2)      | | Config Server  |
   +------------------+ +------------------+ +----------------+
```

### Caracteristiques

- **Deploiement** : 8 microservices dans des containers Docker/Kubernetes
- **Base de donnees** : 8 instances PostgreSQL independantes (database-per-service)
- **Messaging** : Apache Kafka pour les evenements asynchrones entre services
- **Service Discovery** : Eureka pour l'enregistrement et la decouverte dynamique
- **Configuration** : Config Server centralise avec profils par environnement
- **Communication inter-services** : OpenFeign via HTTP REST
- **Resilience** : Circuit Breaker, Retry, Rate Limiter (Resilience4j)
- **Scalabilite** : chaque service peut etre scale independamment
- **Monitoring** : Prometheus + Grafana, ELK Stack, Zipkin

### Avantages

- Scalabilite horizontale par service
- Deploiement independant de chaque service
- Isolation des pannes (un service en echec n'affecte pas les autres)
- Mise a jour sans interruption (rolling updates)
- Adapte aux equipes distribuees (chaque equipe possede son service)

### Inconvenients

- Complexite operationnelle accrue
- Necessite une infrastructure robuste (Kubernetes, monitoring)
- Latence inter-services (appels reseau)
- Coherence eventuelle des donnees (pas de transactions distribuees)

### Profil Spring : `docker`

```yaml
spring:
  profiles:
    active: docker
  cloud:
    config:
      uri: http://config-server:8888
eureka:
  client:
    service-url:
      defaultZone: http://discovery-server:8761/eureka
```

---

## Mode Monolithe (Hopital local)

Le mode monolithe est concu pour les hopitaux locaux disposant d'une infrastructure limitee. Un seul serveur suffit pour faire fonctionner l'ensemble du systeme.

### Architecture

```
+----------------------------------------------------------+
|                   rofecare-server.jar                      |
|                                                            |
|  +--------+ +--------+ +--------+ +--------+ +--------+  |
|  |Identity| |Patient | |Clinical| |MedTech | |Pharmacy|  |
|  +--------+ +--------+ +--------+ +--------+ +--------+  |
|  +--------+ +--------+ +--------+                         |
|  |Finance | |Platform| |Interop |                         |
|  +--------+ +--------+ +--------+                         |
|                                                            |
|  +----------------------------------------------------+   |
|  |         Spring Application Context                  |   |
|  |    (appels directs Java entre modules)              |   |
|  +----------------------------------------------------+   |
|                                                            |
|  +----------------------------------------------------+   |
|  |    Event Bus In-Memory (ApplicationEventPublisher)  |   |
|  +----------------------------------------------------+   |
|                                                            |
+---------------------------+------------------------------+
                            |
                   +--------+--------+
                   |   PostgreSQL    |
                   |  (1 instance)   |
                   |  schemas separes |
                   +-----------------+
```

### Caracteristiques

- **Deploiement** : 1 seul fichier JAR executable (`rofecare-server.jar`)
- **Base de donnees** : 1 instance PostgreSQL avec des schemas separes par module
- **Messaging** : `ApplicationEventPublisher` de Spring (events in-memory)
- **Service Discovery** : desactive (tout est dans le meme processus)
- **Configuration** : fichiers locaux (`application-monolith.yml`)
- **Communication inter-modules** : appels directs Java (injection de dependances)
- **Resilience** : non necessaire (pas d'appels reseau entre modules)

### Avantages

- Simplicite de deploiement et de maintenance
- Performances optimales (pas de latence reseau entre modules)
- Transactions ACID completes (meme base de donnees)
- Infrastructure minimale requise (1 serveur + 1 PostgreSQL)
- Cout operationnel reduit

### Inconvenients

- Pas de scalabilite horizontale par module
- Mise a jour necessitant un arret du systeme
- Point unique de defaillance
- Toutes les equipes travaillent sur le meme artefact de deploiement

### Profil Spring : `monolith`

```yaml
spring:
  profiles:
    active: monolith
  datasource:
    url: jdbc:postgresql://localhost:5432/rofecare
eureka:
  client:
    enabled: false
```

### Module aggregateur : `rofecare-server`

Le module `rofecare-server` est le point d'entree du mode monolithe. Il importe tous les modules metier en tant que dependances Maven et les configure pour fonctionner dans un seul contexte Spring.

```xml
<dependencies>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-identity-service</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-patient-service</artifactId>
    </dependency>
    <!-- ... tous les modules metier ... -->
</dependencies>
```

---

## Mode Desktop (Tauri)

Le mode desktop est concu pour les postes autonomes dans les centres de sante ruraux ou les environnements sans connectivite Internet fiable. Il offre un fonctionnement **100% hors-ligne**.

### Architecture

```
+----------------------------------------------------------+
|                    Application Tauri                       |
|                                                            |
|  +----------------------------------------------------+   |
|  |              Tauri Shell (Rust)                      |   |
|  |         WebView2 (interface utilisateur)            |   |
|  +----------------------------------------------------+   |
|                          |                                 |
|  +----------------------------------------------------+   |
|  |              Nuxt 3 Frontend (SSG)                  |   |
|  |          Pages pre-rendues statiquement             |   |
|  +----------------------------------------------------+   |
|                          |                                 |
|  +----------------------------------------------------+   |
|  |              JRE 21 (Sidecar)                       |   |
|  |          rofecare-server.jar (monolithe)            |   |
|  +----------------------------------------------------+   |
|                          |                                 |
|  +----------------------------------------------------+   |
|  |          PostgreSQL embarque (Sidecar)              |   |
|  +----------------------------------------------------+   |
|                                                            |
|  +----------------------------------------------------+   |
|  |          Module de synchronisation                  |   |
|  |     (sync vers le cloud quand connecte)             |   |
|  +----------------------------------------------------+   |
+----------------------------------------------------------+
```

### Caracteristiques

- **Interface** : Tauri 2.x (Rust + WebView2) avec frontend Nuxt 3 pre-rendu (SSG)
- **Backend** : `rofecare-server.jar` lance en tant que processus sidecar
- **Runtime** : JRE 21 embarque dans l'application
- **Base de donnees** : PostgreSQL embarque, lance comme sidecar
- **Fonctionnement** : 100% hors-ligne, aucune dependance reseau
- **Synchronisation** : module dedie pour synchroniser avec le cloud lorsque la connectivite est disponible

### Avantages

- Fonctionne sans Internet
- Installation simple sur un poste Windows, macOS ou Linux
- Donnees stockees localement (conformite reglementaire)
- Synchronisation opportuniste avec le cloud
- Adapte aux centres de sante ruraux et isoles

### Inconvenients

- Ressources limitees par le materiel du poste
- Donnees non a jour en temps reel (synchronisation periodique)
- Gestion des conflits lors de la synchronisation
- Mise a jour manuelle de l'application

### Processus de demarrage

1. L'utilisateur lance l'application Tauri
2. Tauri demarre PostgreSQL en tant que sidecar
3. Tauri demarre le JRE 21 avec `rofecare-server.jar` en sidecar
4. Le backend applique les migrations Flyway
5. L'interface Nuxt se connecte au backend local
6. Si une connexion Internet est detectee, la synchronisation demarre en arriere-plan

---

## Tableau comparatif

| Aspect                  | Microservices (Cloud)       | Monolithe (Local)           | Desktop (Tauri)             |
|-------------------------|-----------------------------|-----------------------------|-----------------------------|
| **Deploiement**         | 8+ containers               | 1 JAR                       | 1 executable                |
| **Base de donnees**     | 8 instances PostgreSQL      | 1 instance PostgreSQL       | PostgreSQL embarque         |
| **Evenements**          | Apache Kafka                | In-Memory (Spring Events)   | In-Memory (Spring Events)   |
| **Service Discovery**   | Eureka                      | Desactive                   | Desactive                   |
| **Inter-services**      | HTTP REST (Feign)           | Appels Java directs         | Appels Java directs         |
| **Configuration**       | Config Server               | Fichiers locaux             | Fichiers embarques          |
| **Profil Spring**       | `docker`                    | `monolith`                  | `monolith`                  |
| **Scalabilite**         | Horizontale par service     | Verticale uniquement        | Limitee au poste            |
| **Haute disponibilite** | Oui (replicas, LB)          | Non (SPOF)                  | Non (poste unique)          |
| **Connectivite**        | Requise (cloud)             | LAN suffisant               | Aucune requise              |
| **Transactions**        | Coherence eventuelle        | ACID completes              | ACID completes              |
| **Monitoring**          | Prometheus, Grafana, ELK    | Logs locaux, Actuator       | Logs locaux                 |
| **Infrastructure**      | Kubernetes, Docker          | 1 serveur Linux             | 1 poste Windows/macOS/Linux |
| **Cible**               | Grands hopitaux, reseaux    | Hopitaux moyens             | Centres ruraux, postes isoles|
| **Synchronisation**     | Source de verite             | Bidirectionnelle avec cloud | Bidirectionnelle avec cloud |

---

## Mecanisme de synchronisation

La synchronisation permet aux deployments locaux (monolithe et desktop) de rester coherents avec le cloud. Elle est **bidirectionnelle** et fonctionne de maniere **opportuniste** (quand la connexion est disponible).

### Architecture de synchronisation

```
+---------------------+                    +---------------------+
|    Site Local        |                    |    Cloud            |
|  (Monolithe/Desktop) |                    |  (Microservices)    |
|                     |                    |                     |
|  +---------------+  |                    |  +---------------+  |
|  | Application   |  |                    |  | Application   |  |
|  +-------+-------+  |                    |  +-------+-------+  |
|          |           |                    |          |           |
|  +-------+-------+  |     HTTPS/REST     |  +-------+-------+  |
|  | SyncPush      +--+----- Push -------->+--+ SyncPull      |  |
|  | Service       |  |                    |  | Service       |  |
|  +---------------+  |                    |  +---------------+  |
|                     |                    |                     |
|  +---------------+  |                    |  +---------------+  |
|  | SyncPull      +<-+----- Pull --------+--+ SyncPush      |  |
|  | Service       |  |                    |  | Service       |  |
|  +---------------+  |                    |  +---------------+  |
|          ^           |                    |          ^           |
|          |           |                    |          |           |
|  +-------+-------+  |                    |  +-------+-------+  |
|  | SyncOutbox    |  |                    |  | SyncOutbox    |  |
|  | Entry         |  |                    |  | Entry         |  |
|  +-------+-------+  |                    |  +-------+-------+  |
|          ^           |                    |          ^           |
|          |           |                    |          |           |
|  +-------+-------+  |                    |  +-------+-------+  |
|  | SyncChange    |  |                    |  | SyncChange    |  |
|  | Capture       |  |                    |  | Capture       |  |
|  +---------------+  |                    |  +---------------+  |
+---------------------+                    +---------------------+
```

### Composants

#### SyncChangeCapture

Listener Hibernate qui intercepte automatiquement les modifications de donnees (INSERT, UPDATE, DELETE) et cree des entrees dans la table outbox.

```java
// Intercepte les changements Hibernate
@Component
public class SyncChangeCapture implements PostInsertEventListener,
                                          PostUpdateEventListener,
                                          PostDeleteEventListener {
    // Capture automatique des changements
    // Cree une SyncOutboxEntry pour chaque modification
}
```

#### SyncOutboxEntry

Table d'outbox stockant les changements en attente de synchronisation. Chaque entree contient :

| Champ         | Description                                  |
|---------------|----------------------------------------------|
| `id`          | Identifiant unique de l'entree               |
| `entityType`  | Type de l'entite modifiee                    |
| `entityId`    | Identifiant de l'entite                      |
| `operation`   | Type d'operation (INSERT, UPDATE, DELETE)     |
| `payload`     | Donnees serialisees (JSON)                   |
| `nodeId`      | Identifiant du noeud source                  |
| `createdAt`   | Horodatage de la modification                |
| `syncedAt`    | Horodatage de la synchronisation (null si en attente) |
| `version`     | Version de l'entite pour la detection de conflits |

#### SyncPushService

Envoie les changements locaux vers le cloud. Fonctionne en batch pour optimiser les performances.

#### SyncPullService

Recoit les changements du cloud et les applique localement. Gere la detection et la resolution de conflits.

### Resolution de conflits : Last-Write-Wins

En cas de modification concurrente de la meme entite sur deux noeuds differents, la strategie **Last-Write-Wins (LWW)** est appliquee :

```
1. Comparer updatedAt → la modification la plus recente gagne
2. Si egalite : comparer version → la version la plus elevee gagne
3. Si egalite : comparer nodeId → le noeud avec l'ID le plus eleve gagne (deterministe)
```

```
Noeud A modifie Patient #123 a 10:05:30 (version 5)
Noeud B modifie Patient #123 a 10:05:31 (version 4)

Resolution : Noeud B gagne (updatedAt plus recent)
```

### Flux de synchronisation

1. **Detection** : `SyncChangeCapture` intercepte une modification Hibernate
2. **Enregistrement** : une `SyncOutboxEntry` est creee dans la meme transaction
3. **Planification** : un scheduler periodique declenche le push des entrees non synchronisees
4. **Push** : `SyncPushService` envoie un batch de modifications au cloud via HTTPS
5. **Reception** : le cloud recoit et applique les modifications (avec resolution de conflits)
6. **Pull** : `SyncPullService` recupere les modifications du cloud non encore presentes localement
7. **Application** : les modifications distantes sont appliquees localement
8. **Acquittement** : les entrees outbox sont marquees comme synchronisees

---

## Matrice de decision

Utilisez cette matrice pour determiner le mode de deploiement adapte a votre contexte.

### Criteres de choix

| Critere                          | Microservices | Monolithe | Desktop |
|----------------------------------|:---:|:---:|:---:|
| Internet haut debit disponible   | Requis | Optionnel | Non requis |
| Equipe IT dediee                 | Requis | Recommande | Non requis |
| Plus de 500 utilisateurs         | Recommande | Possible | Non adapte |
| Moins de 50 utilisateurs         | Surdimensionne | Adapte | Adapte |
| Haute disponibilite requise      | Adapte | Non adapte | Non adapte |
| Budget infrastructure limite     | Non adapte | Adapte | Adapte |
| Centre de sante rural            | Non adapte | Possible | Recommande |
| Reseau d'hopitaux                | Recommande | Par site | Par poste |
| Conformite donnees locales       | Cloud souverain | Adapte | Adapte |
| Synchronisation multi-sites      | Natif | Via sync | Via sync |

### Scenarios recommandes

**Scenario 1 : Grand hopital urbain**
- Mode : **Microservices** sur cloud prive ou public
- Justification : haute disponibilite, scalabilite, equipe IT dediee

**Scenario 2 : Hopital moyen en zone urbaine**
- Mode : **Monolithe** sur serveur local
- Justification : simplicite, cout reduit, Internet non garanti

**Scenario 3 : Centre de sante rural**
- Mode : **Desktop** sur un poste
- Justification : pas d'Internet fiable, pas d'equipe IT, autonomie complete

**Scenario 4 : Reseau d'hopitaux (mixte)**
- Mode : **Microservices** (central) + **Monolithe** ou **Desktop** (sites peripheriques)
- Justification : source de verite centralisee, autonomie des sites, synchronisation

---

## Configuration par mode

### Profils Spring

Les differences de comportement entre les modes sont gerees par les profils Spring Boot.

#### Profil `docker` (Microservices)

```yaml
spring:
  profiles:
    active: docker
  datasource:
    url: jdbc:postgresql://${SERVICE_NAME}-db:5432/rofecare_${SERVICE_NAME}
  kafka:
    bootstrap-servers: kafka:9092
  cloud:
    config:
      uri: http://config-server:8888
  redis:
    host: redis
    port: 6379

eureka:
  client:
    enabled: true
    service-url:
      defaultZone: http://discovery-server:8761/eureka
  instance:
    prefer-ip-address: true

resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
  retry:
    configs:
      default:
        max-attempts: 3
        wait-duration: 1s
```

#### Profil `monolith` (Monolithe / Desktop)

```yaml
spring:
  profiles:
    active: monolith
  datasource:
    url: jdbc:postgresql://localhost:5432/rofecare
  kafka:
    enabled: false
  cloud:
    config:
      enabled: false

eureka:
  client:
    enabled: false

# Events in-memory via Spring ApplicationEventPublisher
rofecare:
  events:
    type: in-memory
  sync:
    enabled: true
    push-interval: 300  # secondes
    pull-interval: 300  # secondes
    cloud-url: https://cloud.rofecare.com/api/sync
```

### Differences de configuration cles

| Parametre                | Microservices              | Monolithe / Desktop         |
|--------------------------|----------------------------|-----------------------------|
| `spring.datasource.url`  | Par service (DB dediee)    | Unique (schemas multiples)  |
| `spring.kafka.enabled`   | `true`                     | `false`                     |
| `eureka.client.enabled`  | `true`                     | `false`                     |
| `spring.cloud.config`    | Config Server distant      | Desactive                   |
| `rofecare.events.type`   | `kafka`                    | `in-memory`                 |
| `rofecare.sync.enabled`  | `false` (source de verite) | `true` (sync avec cloud)    |
| `resilience4j`           | Active (appels reseau)     | Desactive (appels locaux)   |

### Adaptation du code

L'architecture hexagonale permet d'adapter le comportement sans modifier la logique metier :

```java
// Adapter pour les evenements - interface commune
public interface DomainEventPublisher {
    void publish(DomainEvent event);
}

// Implementation Kafka (microservices)
@Profile("docker")
@Component
public class KafkaDomainEventPublisher implements DomainEventPublisher {
    // Publication via Kafka
}

// Implementation In-Memory (monolithe/desktop)
@Profile("monolith")
@Component
public class InMemoryDomainEventPublisher implements DomainEventPublisher {
    // Publication via ApplicationEventPublisher
}
```

```java
// Adapter pour les appels inter-services
public interface PatientServicePort {
    PatientInfo getPatient(UUID patientId);
}

// Implementation Feign (microservices)
@Profile("docker")
@FeignClient("patient-service")
public interface PatientFeignClient extends PatientServicePort {}

// Implementation directe (monolithe/desktop)
@Profile("monolith")
@Component
public class PatientDirectAdapter implements PatientServicePort {
    // Appel direct au service Java
}
```

---

*Voir aussi :*
- [Vue d'ensemble de l'architecture](overview.md)
- [Sous-domaines et Bounded Contexts](subdomains.md)
- [Guide de deploiement](../deployment/)
