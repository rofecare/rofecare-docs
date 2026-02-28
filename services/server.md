# Monolith Aggregator (rofecare-server)

> **Port** : 8080 (mode standalone)
> **Base de donnees** : `rofecare` (PostgreSQL unique)
> **Module Maven** : `rofecare-server`
> **Spring Profile** : `monolith`
> **Package racine** : `com.rofecare.server`

## Vue d'ensemble

`rofecare-server` est le module d'aggregation monolithique de Rofecare. Il combine les 8 services metier (Identity, Patient, Clinical, MedTech, Pharmacy, Finance, Platform, Interoperability) en un seul JAR deployable, partageant une base de donnees PostgreSQL unique.

Ce mode de deploiement est concu pour les etablissements de sante qui :
- Ne disposent pas d'une infrastructure Kubernetes/Docker
- Preferent une installation sur site (on-premise) simplifiee
- Ont des contraintes de bande passante reseau
- Souhaitent un deploiement et une maintenance minimaux

Le monolithe maintient une synchronisation bidirectionnelle avec l'instance cloud, permettant un fonctionnement hybride ou le site local fonctionne de maniere autonome tout en synchronisant les donnees avec le cloud.

## Architecture

```
rofecare-server/
├── src/main/java/com/rofecare/server/
│   ├── RofecareServerApplication.java    # Point d'entree principal
│   ├── config/
│   │   ├── MonolithConfig.java           # Configuration du mode monolithe
│   │   ├── InMemoryEventConfig.java      # Remplacement Kafka -> in-memory
│   │   └── DirectServiceClientConfig.java # Remplacement Feign -> appels directs
│   ├── event/
│   │   └── InMemoryEventPublisher.java   # Implementation EventPublisher en memoire
│   ├── client/
│   │   ├── DirectPatientClient.java      # Client direct vers PatientService
│   │   ├── DirectClinicalClient.java     # Client direct vers ClinicalService
│   │   └── ...                           # Un client direct par service
│   └── sync/
│       ├── SyncChangeCapture.java        # Capture des modifications locales
│       ├── SyncOutboxEntry.java          # Entree de la outbox de synchronisation
│       ├── SyncPushService.java          # Envoi des modifications vers le cloud
│       ├── SyncPullService.java          # Reception des modifications depuis le cloud
│       └── SyncConflictResolver.java     # Resolution des conflits de synchronisation
├── src/main/resources/
│   ├── application.yml                   # Configuration de base
│   └── application-monolith.yml          # Configuration specifique au monolithe
└── pom.xml                               # Inclut tous les modules comme dependances
```

## Dependances Maven

Le module `rofecare-server` inclut tous les services comme dependances Maven :

```xml
<dependencies>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-common</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-identity</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-patient</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-clinical</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-medtech</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-pharmacy</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-finance</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-platform</artifactId>
    </dependency>
    <dependency>
        <groupId>com.rofecare</groupId>
        <artifactId>rofecare-interoperability</artifactId>
    </dependency>
</dependencies>
```

## Remplacement d'infrastructure

En mode monolithe, certains composants d'infrastructure sont remplaces par des alternatives legeres :

| Composant microservices | Remplacement monolithe | Raison |
|------------------------|----------------------|--------|
| **Apache Kafka** | `InMemoryEventPublisher` | Les evenements sont dispatches en memoire via Spring `ApplicationEventPublisher`. Pas besoin de broker externe. |
| **OpenFeign clients** | `DirectServiceClient` | Les appels inter-services deviennent des appels de methodes Java directs. Pas de latence reseau. |
| **Eureka Discovery** | Desactive | Un seul processus, pas besoin de decouverte de services. |
| **Config Server** | Desactive | Configuration locale via `application-monolith.yml`. |
| **Bases de donnees multiples** | PostgreSQL unique (`rofecare`) | Tous les schemas cohabitent dans une seule base, chaque service utilisant un prefixe de table ou un schema dedie. |

### InMemoryEventPublisher

```java
@Component
@Profile("monolith")
public class InMemoryEventPublisher implements EventPublisher {

    private final ApplicationEventPublisher springEventPublisher;

    @Override
    public void publish(String topic, Object event) {
        springEventPublisher.publishEvent(new InMemoryDomainEvent(topic, event));
    }
}
```

### Direct Service Clients

```java
@Component
@Profile("monolith")
public class DirectPatientClient implements PatientFeignClient {

    private final PatientApplicationService patientService;

    @Override
    public PatientDto getPatient(UUID patientId) {
        // Appel direct en memoire, sans passage par le reseau
        return patientService.findById(patientId);
    }
}
```

## Mecanisme de synchronisation

Le mecanisme de synchronisation bidirectionnelle permet au monolithe local de fonctionner de maniere autonome tout en maintenant la coherence des donnees avec l'instance cloud.

### Composants

#### SyncChangeCapture

Intercepte les modifications apportees aux entites JPA via un `EntityListener`. Chaque creation, modification ou suppression est enregistree dans la table `sync_outbox`.

```java
@EntityListeners(SyncChangeCapture.class)
public abstract class AuditableEntity {
    // ...
}
```

#### SyncOutboxEntry

Represente une modification en attente de synchronisation :

| Champ | Type | Description |
|-------|------|-------------|
| `id` | `UUID` | Identifiant unique de l'entree |
| `entityType` | `String` | Type de l'entite modifiee (ex: `Patient`, `Invoice`) |
| `entityId` | `UUID` | ID de l'entite modifiee |
| `operation` | `Enum` | Type d'operation : `INSERT`, `UPDATE`, `DELETE` |
| `payload` | `JSONB` | Donnees serialisees de l'entite |
| `nodeId` | `String` | Identifiant du noeud source |
| `syncVersion` | `Long` | Version de synchronisation |
| `createdAt` | `Instant` | Date/heure de la modification |
| `synced` | `Boolean` | Indique si l'entree a ete synchronisee |

#### SyncPushService

Service planifie qui pousse les modifications locales vers le cloud :

1. Lit les entrees non synchronisees depuis `sync_outbox` (batch)
2. Envoie les modifications au cloud via API REST
3. Marque les entrees comme synchronisees en cas de succes
4. En cas d'echec, reessaie au prochain cycle

#### SyncPullService

Service planifie qui tire les modifications depuis le cloud :

1. Lit le dernier checkpoint depuis `sync_checkpoint`
2. Interroge le cloud pour les modifications depuis ce checkpoint
3. Applique les modifications localement
4. Met a jour le checkpoint

#### SyncConflictResolver

Resout les conflits lorsque la meme entite a ete modifiee simultanement sur le noeud local et le cloud.

### Strategie de resolution des conflits

La resolution utilise l'approche **Last-Write-Wins (LWW)** avec la hierarchie suivante :

```
1. updatedAt    -> La modification la plus recente gagne
2. syncVersion  -> En cas d'egalite, la version de sync la plus elevee gagne
3. nodeId       -> En dernier recours, l'identifiant du noeud est utilise comme tie-breaker
```

```java
public class SyncConflictResolver {

    public <T extends AuditableEntity> T resolve(T local, T remote) {
        // 1. Comparaison par updatedAt
        if (remote.getUpdatedAt().isAfter(local.getUpdatedAt())) {
            return remote;
        }
        if (local.getUpdatedAt().isAfter(remote.getUpdatedAt())) {
            return local;
        }

        // 2. Comparaison par syncVersion
        if (remote.getSyncVersion() > local.getSyncVersion()) {
            return remote;
        }
        if (local.getSyncVersion() > remote.getSyncVersion()) {
            return local;
        }

        // 3. Tie-breaker par nodeId
        return remote.getNodeId().compareTo(local.getNodeId()) > 0
            ? remote : local;
    }
}
```

## Tables de synchronisation

### sync_outbox

```sql
CREATE TABLE sync_outbox (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type   VARCHAR(100) NOT NULL,
    entity_id     UUID NOT NULL,
    operation     VARCHAR(10) NOT NULL,    -- INSERT, UPDATE, DELETE
    payload       JSONB NOT NULL,
    node_id       VARCHAR(50) NOT NULL,
    sync_version  BIGINT NOT NULL,
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    synced        BOOLEAN NOT NULL DEFAULT FALSE,
    synced_at     TIMESTAMP,
    INDEX idx_sync_outbox_unsynced (synced, created_at)
);
```

### sync_checkpoint

```sql
CREATE TABLE sync_checkpoint (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  VARCHAR(50) NOT NULL UNIQUE,
    last_sync_version BIGINT NOT NULL DEFAULT 0,
    last_synced_at  TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## Configuration

```yaml
# application-monolith.yml
spring:
  profiles:
    active: monolith

  datasource:
    url: jdbc:postgresql://localhost:5432/rofecare
    username: ${DB_USERNAME:rofecare}
    password: ${DB_PASSWORD:rofecare}
  jpa:
    hibernate:
      ddl-auto: validate

  # Kafka desactive en mode monolithe
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration

# Eureka desactive
eureka:
  client:
    enabled: false

# Configuration de la synchronisation
sync:
  enabled: ${SYNC_ENABLED:true}
  cloud-url: ${CLOUD_API_URL:https://api.rofecare.com}
  cloud-api-key: ${CLOUD_API_KEY}
  node-id: ${NODE_ID:local-node-1}
  push:
    interval: 30s                    # Frequence d'envoi des modifications
    batch-size: 100                  # Nombre maximal d'entrees par batch
  pull:
    interval: 30s                    # Frequence de recuperation des modifications
    batch-size: 100                  # Nombre maximal d'entrees par batch
  conflict-resolution:
    strategy: last-write-wins        # Strategie de resolution des conflits

server:
  port: 8080
```

## Demarrage

### Mode monolithe standalone

```bash
# Via Maven
mvn -pl rofecare-server spring-boot:run -Dspring.profiles.active=monolith

# Via JAR
java -jar rofecare-server.jar --spring.profiles.active=monolith

# Variables d'environnement requises
export DB_USERNAME=rofecare
export DB_PASSWORD=rofecare
export NODE_ID=site-kinshasa-01
export CLOUD_API_URL=https://api.rofecare.com
export CLOUD_API_KEY=your-api-key
```

### Docker

```bash
docker run -d \
  --name rofecare-server \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=monolith \
  -e DB_USERNAME=rofecare \
  -e DB_PASSWORD=rofecare \
  -e NODE_ID=site-kinshasa-01 \
  -e CLOUD_API_URL=https://api.rofecare.com \
  -e CLOUD_API_KEY=your-api-key \
  rofecare/server:latest
```

## Quand utiliser le mode monolithe

| Scenario | Recommandation |
|----------|---------------|
| Etablissement avec connexion internet stable et infrastructure cloud | Mode **microservices** (deploiement Kubernetes) |
| Etablissement avec connexion intermittente | Mode **monolithe** avec synchronisation |
| Petit etablissement / cabinet medical | Mode **monolithe** standalone |
| Phase de developpement et tests | Mode **monolithe** (plus simple a demarrer) |
| Deploiement sur site sans equipe IT | Mode **monolithe** avec Docker Compose |

## Decisions architecturales

1. **Profil Spring conditionnel** : Le profil `monolith` active les beans de remplacement (`InMemoryEventPublisher`, `DirectServiceClient`) via `@Profile("monolith")`, sans modifier le code des services individuels.

2. **Base de donnees unique** : En mode monolithe, tous les services partagent la meme base PostgreSQL. Les tables sont prefixees ou organisees en schemas pour eviter les conflits de nommage.

3. **Synchronisation outbox pattern** : Le pattern Transactional Outbox garantit qu'aucune modification n'est perdue meme en cas de panne du reseau. Les modifications sont d'abord ecrites dans `sync_outbox` (dans la meme transaction), puis envoyees au cloud de maniere asynchrone.

4. **Last-Write-Wins** : La strategie de resolution des conflits est deliberement simple (LWW) car les cas de modification simultanee de la meme entite depuis deux noeuds sont rares dans le contexte hospitalier (chaque patient est generalement suivi par un seul site).

5. **Synchronisation configurable** : L'intervalle de synchronisation (30s par defaut) et la taille des batchs (100 par defaut) sont configurables pour s'adapter a la bande passante et au volume de donnees de chaque site.

6. **Fonctionnement offline** : Le monolithe continue de fonctionner normalement meme sans connexion au cloud. Les modifications sont accumulees dans `sync_outbox` et synchronisees des que la connexion est retablie.
