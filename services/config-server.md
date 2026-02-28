# Config Server

> **Module** : `rofecare-config-server`
> **Port** : `8888`
> **Framework** : Spring Cloud Config Server

## Vue d'ensemble

Le Config Server centralise la gestion de la configuration de tous les microservices de la plateforme Rofecare. Chaque service recupere sa configuration au demarrage depuis ce serveur, garantissant une coherence globale et facilitant les changements de configuration sans redeploiement.

Le serveur utilise le profil **native** (configuration basee sur des fichiers locaux) plutot qu'un depot Git distant.

## Fonctionnement

### Demarrage

1. Le Config Server demarre en premier (avant tous les autres services).
2. Il charge les fichiers de configuration depuis `src/main/resources/config/`.
3. Chaque microservice, au demarrage, contacte le Config Server a l'adresse `http://localhost:8888` pour recuperer sa configuration.
4. Le service applique la configuration recue, qui surcharge ses proprietes locales.

### Resolution de configuration

Pour un service donne (par exemple `rofecare-service-identity`), le Config Server fusionne :

1. `application.yml` : configuration partagee (commune a tous les services).
2. `rofecare-service-identity.yml` : configuration specifique au service.

Les proprietes specifiques surchargent les proprietes partagees en cas de conflit.

## Structure des fichiers de configuration

```
rofecare-config-server/
└── src/main/resources/
    └── config/
        ├── application.yml                          # Configuration partagee
        ├── rofecare-service-identity.yml             # Identity Service
        ├── rofecare-service-patient.yml               # Patient Service
        ├── rofecare-service-clinical.yml              # Clinical Service
        ├── rofecare-service-medical-technology.yml    # Medical Technology Service
        ├── rofecare-service-pharmacy.yml              # Pharmacy Service
        ├── rofecare-service-finance.yml               # Finance Service
        ├── rofecare-service-platform.yml              # Platform Service
        └── rofecare-service-interoperability.yml      # Interoperability Service
```

## Configuration partagee (application.yml)

Le fichier `application.yml` contient les proprietes communes a tous les services :

### Eureka Client

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
    fetch-registry: true
    register-with-eureka: true
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10
```

### JPA / Hibernate

```yaml
spring:
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: update
    show-sql: false
    properties:
      hibernate:
        format_sql: true
```

Configuration pour PostgreSQL avec mise a jour automatique du schema (`ddl-auto: update`).

### Redis

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
```

Utilise pour le cache, les sessions et le rate limiting du gateway.

### Kafka

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      acks: all
    consumer:
      auto-offset-reset: earliest
```

Bus de messages pour la communication asynchrone entre services.

### Resilience4j

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

Configuration par defaut des circuit breakers : ouverture a 50% d'echec, attente de 10 secondes avant tentative de fermeture.

## Configuration specifique par service

Chaque fichier `rofecare-service-{name}.yml` contient les proprietes propres a un service :

| Propriete | Description | Exemple |
|---|---|---|
| `server.port` | Port d'ecoute du service | `8081` |
| `spring.datasource.url` | URL de la base de donnees PostgreSQL | `jdbc:postgresql://localhost:5432/rofecare_identity` |
| `spring.datasource.username` | Utilisateur de la base | `postgres` |
| `spring.datasource.password` | Mot de passe de la base | `${DB_PASSWORD}` |
| Proprietes metier | Configuration specifique au domaine | Variables selon le service |

## Variables d'environnement

Les fichiers de configuration utilisent des variables d'environnement avec des valeurs par defaut pour faciliter le deploiement :

| Variable | Description | Valeur par defaut |
|---|---|---|
| `REDIS_HOST` | Adresse du serveur Redis | `localhost` |
| `REDIS_PORT` | Port Redis | `6379` |
| `REDIS_PASSWORD` | Mot de passe Redis | _(vide)_ |
| `KAFKA_BOOTSTRAP_SERVERS` | Adresse du broker Kafka | `localhost:9092` |
| `DB_PASSWORD` | Mot de passe des bases PostgreSQL | Variable selon l'environnement |
| `EUREKA_SERVER_URL` | URL du Discovery Server | `http://localhost:8761/eureka/` |

## Recuperation de configuration par les services

Chaque microservice declare sa dependance au Config Server dans son fichier `application.yml` ou `bootstrap.yml` :

```yaml
spring:
  config:
    import: optional:configserver:http://localhost:8888
  application:
    name: rofecare-service-identity
```

Le `application.name` determine quel fichier de configuration specifique sera charge depuis le Config Server.

## Rafraichissement de configuration

Les services exposent un endpoint Actuator pour rafraichir leur configuration a chaud, sans redemarrage :

```bash
POST http://localhost:{port}/actuator/refresh
```

Cette requete force le service a recharger sa configuration depuis le Config Server et a mettre a jour les beans annotes `@RefreshScope`.

## Configuration du serveur

```yaml
server:
  port: 8888

spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          search-locations: classpath:/config
```

Le profil `native` indique que les fichiers de configuration sont stockes localement (dans le classpath) plutot que dans un depot Git distant.
