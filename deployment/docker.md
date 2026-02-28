# Reference Docker Compose

Documentation complete des fichiers Docker Compose, images, volumes et reseaux utilises par Rofecare HIS.

---

## Table des matieres

- [Fichiers Docker Compose](#fichiers-docker-compose)
- [docker-compose.yml (core)](#docker-composeyml-core)
- [docker-compose.prod.yml (production overlay)](#docker-composeprodym-production-overlay)
- [docker-compose.monitoring.yml (monitoring stack)](#docker-composemonitoringyml-monitoring-stack)
- [Variables d'environnement (.env)](#variables-denvironnement-env)
- [Images Docker utilisees](#images-docker-utilisees)
- [Dockerfile multi-stage build](#dockerfile-multi-stage-build)
- [Gestion des volumes](#gestion-des-volumes)
- [Configuration reseau](#configuration-reseau)
- [Commandes Docker courantes](#commandes-docker-courantes)

---

## Fichiers Docker Compose

Le projet utilise une approche multi-fichiers pour separer les preoccupations :

| Fichier                          | Usage                             |
| -------------------------------- | --------------------------------- |
| `docker-compose.yml`            | Configuration de base (dev/test)  |
| `docker-compose.prod.yml`       | Overlay pour la production        |
| `docker-compose.monitoring.yml` | Stack de monitoring               |

### Combinaison de fichiers

```bash
# Developpement (defaut)
docker compose up -d

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Production + monitoring
docker compose -f docker-compose.yml \
  -f docker-compose.prod.yml \
  -f docker-compose.monitoring.yml up -d
```

---

## docker-compose.yml (core)

Fichier principal contenant la definition de tous les services :

```yaml
version: "3.8"

services:
  # ================================================================
  # INFRASTRUCTURE - Bases de donnees
  # ================================================================
  identity-db:
    image: postgres:16-alpine
    container_name: identity-db
    environment:
      POSTGRES_DB: ${IDENTITY_DB_NAME:-identity_db}
      POSTGRES_USER: ${IDENTITY_DB_USER:-identity_user}
      POSTGRES_PASSWORD: ${IDENTITY_DB_PASSWORD:-password}
    ports:
      - "5433:5432"
    volumes:
      - identity-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${IDENTITY_DB_USER:-identity_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  patient-db:
    image: postgres:16-alpine
    container_name: patient-db
    environment:
      POSTGRES_DB: ${PATIENT_DB_NAME:-patient_db}
      POSTGRES_USER: ${PATIENT_DB_USER:-patient_user}
      POSTGRES_PASSWORD: ${PATIENT_DB_PASSWORD:-password}
    ports:
      - "5434:5432"
    volumes:
      - patient-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PATIENT_DB_USER:-patient_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  clinical-db:
    image: postgres:16-alpine
    container_name: clinical-db
    environment:
      POSTGRES_DB: ${CLINICAL_DB_NAME:-clinical_db}
      POSTGRES_USER: ${CLINICAL_DB_USER:-clinical_user}
      POSTGRES_PASSWORD: ${CLINICAL_DB_PASSWORD:-password}
    ports:
      - "5435:5432"
    volumes:
      - clinical-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${CLINICAL_DB_USER:-clinical_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  medtech-db:
    image: postgres:16-alpine
    container_name: medtech-db
    environment:
      POSTGRES_DB: ${MEDTECH_DB_NAME:-medtech_db}
      POSTGRES_USER: ${MEDTECH_DB_USER:-medtech_user}
      POSTGRES_PASSWORD: ${MEDTECH_DB_PASSWORD:-password}
    ports:
      - "5436:5432"
    volumes:
      - medtech-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${MEDTECH_DB_USER:-medtech_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  pharmacy-db:
    image: postgres:16-alpine
    container_name: pharmacy-db
    environment:
      POSTGRES_DB: ${PHARMACY_DB_NAME:-pharmacy_db}
      POSTGRES_USER: ${PHARMACY_DB_USER:-pharmacy_user}
      POSTGRES_PASSWORD: ${PHARMACY_DB_PASSWORD:-password}
    ports:
      - "5437:5432"
    volumes:
      - pharmacy-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PHARMACY_DB_USER:-pharmacy_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  finance-db:
    image: postgres:16-alpine
    container_name: finance-db
    environment:
      POSTGRES_DB: ${FINANCE_DB_NAME:-finance_db}
      POSTGRES_USER: ${FINANCE_DB_USER:-finance_user}
      POSTGRES_PASSWORD: ${FINANCE_DB_PASSWORD:-password}
    ports:
      - "5438:5432"
    volumes:
      - finance-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${FINANCE_DB_USER:-finance_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  platform-db:
    image: postgres:16-alpine
    container_name: platform-db
    environment:
      POSTGRES_DB: ${PLATFORM_DB_NAME:-platform_db}
      POSTGRES_USER: ${PLATFORM_DB_USER:-platform_user}
      POSTGRES_PASSWORD: ${PLATFORM_DB_PASSWORD:-password}
    ports:
      - "5439:5432"
    volumes:
      - platform-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${PLATFORM_DB_USER:-platform_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  interop-db:
    image: postgres:16-alpine
    container_name: interop-db
    environment:
      POSTGRES_DB: ${INTEROP_DB_NAME:-interop_db}
      POSTGRES_USER: ${INTEROP_DB_USER:-interop_user}
      POSTGRES_PASSWORD: ${INTEROP_DB_PASSWORD:-password}
    ports:
      - "5440:5432"
    volumes:
      - interop-db-data:/var/lib/postgresql/data
    networks:
      - database
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${INTEROP_DB_USER:-interop_user}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ================================================================
  # INFRASTRUCTURE - Cache et messagerie
  # ================================================================
  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server --requirepass ${REDIS_PASSWORD:-password}
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-password}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - backend
    healthcheck:
      test: ["CMD", "echo", "ruok", "|", "nc", "localhost", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    depends_on:
      zookeeper:
        condition: service_healthy
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    ports:
      - "9092:9092"
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - backend

  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui
    depends_on:
      - kafka
    environment:
      KAFKA_CLUSTERS_0_NAME: rofecare
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    ports:
      - "9000:8080"
    networks:
      - backend

  # ================================================================
  # SPRING CLOUD
  # ================================================================
  config-server:
    build:
      context: ./rofecare-config-server
      dockerfile: Dockerfile
    container_name: config-server
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CLOUD_CONFIG_SERVER_GIT_URI: ${CONFIG_SERVER_GIT_URI}
    ports:
      - "8888:8888"
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8888/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 10

  discovery-server:
    build:
      context: ./rofecare-discovery-server
      dockerfile: Dockerfile
    container_name: discovery-server
    depends_on:
      config-server:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
    ports:
      - "8761:8761"
    networks:
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 10

  gateway:
    build:
      context: ./rofecare-gateway
      dockerfile: Dockerfile
    container_name: gateway
    depends_on:
      discovery-server:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
    ports:
      - "8080:8080"
    networks:
      - backend
      - frontend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 15s
      timeout: 5s
      retries: 10

  # ================================================================
  # SERVICES METIER
  # ================================================================
  identity-service:
    build:
      context: ./rofecare-identity-service
      dockerfile: Dockerfile
    container_name: identity-service
    depends_on:
      discovery-server:
        condition: service_healthy
      identity-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://identity-db:5432/${IDENTITY_DB_NAME:-identity_db}
      SPRING_DATASOURCE_USERNAME: ${IDENTITY_DB_USER:-identity_user}
      SPRING_DATASOURCE_PASSWORD: ${IDENTITY_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      SPRING_DATA_REDIS_HOST: redis
      SPRING_DATA_REDIS_PASSWORD: ${REDIS_PASSWORD:-password}
    ports:
      - "8081:8081"
    networks:
      - backend
      - database

  patient-service:
    build:
      context: ./rofecare-patient-service
      dockerfile: Dockerfile
    container_name: patient-service
    depends_on:
      discovery-server:
        condition: service_healthy
      patient-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://patient-db:5432/${PATIENT_DB_NAME:-patient_db}
      SPRING_DATASOURCE_USERNAME: ${PATIENT_DB_USER:-patient_user}
      SPRING_DATASOURCE_PASSWORD: ${PATIENT_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8082:8082"
    networks:
      - backend
      - database

  clinical-service:
    build:
      context: ./rofecare-clinical-service
      dockerfile: Dockerfile
    container_name: clinical-service
    depends_on:
      discovery-server:
        condition: service_healthy
      clinical-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://clinical-db:5432/${CLINICAL_DB_NAME:-clinical_db}
      SPRING_DATASOURCE_USERNAME: ${CLINICAL_DB_USER:-clinical_user}
      SPRING_DATASOURCE_PASSWORD: ${CLINICAL_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8083:8083"
    networks:
      - backend
      - database

  medical-technology-service:
    build:
      context: ./rofecare-medical-technology-service
      dockerfile: Dockerfile
    container_name: medical-technology-service
    depends_on:
      discovery-server:
        condition: service_healthy
      medtech-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://medtech-db:5432/${MEDTECH_DB_NAME:-medtech_db}
      SPRING_DATASOURCE_USERNAME: ${MEDTECH_DB_USER:-medtech_user}
      SPRING_DATASOURCE_PASSWORD: ${MEDTECH_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8084:8084"
    networks:
      - backend
      - database

  pharmacy-service:
    build:
      context: ./rofecare-pharmacy-service
      dockerfile: Dockerfile
    container_name: pharmacy-service
    depends_on:
      discovery-server:
        condition: service_healthy
      pharmacy-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://pharmacy-db:5432/${PHARMACY_DB_NAME:-pharmacy_db}
      SPRING_DATASOURCE_USERNAME: ${PHARMACY_DB_USER:-pharmacy_user}
      SPRING_DATASOURCE_PASSWORD: ${PHARMACY_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8085:8085"
    networks:
      - backend
      - database

  finance-service:
    build:
      context: ./rofecare-finance-service
      dockerfile: Dockerfile
    container_name: finance-service
    depends_on:
      discovery-server:
        condition: service_healthy
      finance-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://finance-db:5432/${FINANCE_DB_NAME:-finance_db}
      SPRING_DATASOURCE_USERNAME: ${FINANCE_DB_USER:-finance_user}
      SPRING_DATASOURCE_PASSWORD: ${FINANCE_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8086:8086"
    networks:
      - backend
      - database

  platform-service:
    build:
      context: ./rofecare-platform-service
      dockerfile: Dockerfile
    container_name: platform-service
    depends_on:
      discovery-server:
        condition: service_healthy
      platform-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://platform-db:5432/${PLATFORM_DB_NAME:-platform_db}
      SPRING_DATASOURCE_USERNAME: ${PLATFORM_DB_USER:-platform_user}
      SPRING_DATASOURCE_PASSWORD: ${PLATFORM_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8087:8087"
    networks:
      - backend
      - database

  interoperability-service:
    build:
      context: ./rofecare-interoperability-service
      dockerfile: Dockerfile
    container_name: interoperability-service
    depends_on:
      discovery-server:
        condition: service_healthy
      interop-db:
        condition: service_healthy
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_CONFIG_IMPORT: configserver:http://config-server:8888
      SPRING_DATASOURCE_URL: jdbc:postgresql://interop-db:5432/${INTEROP_DB_NAME:-interop_db}
      SPRING_DATASOURCE_USERNAME: ${INTEROP_DB_USER:-interop_user}
      SPRING_DATASOURCE_PASSWORD: ${INTEROP_DB_PASSWORD:-password}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://discovery-server:8761/eureka/
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    ports:
      - "8088:8088"
    networks:
      - backend
      - database

# ================================================================
# VOLUMES
# ================================================================
volumes:
  identity-db-data:
  patient-db-data:
  clinical-db-data:
  medtech-db-data:
  pharmacy-db-data:
  finance-db-data:
  platform-db-data:
  interop-db-data:
  redis-data:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:

# ================================================================
# NETWORKS
# ================================================================
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
  database:
    driver: bridge
```

---

## docker-compose.prod.yml (production overlay)

Voir [server-install.md](./server-install.md#ressources-et-optimisation) pour le contenu detaille de ce fichier. En resume, il ajoute :

- `restart: always` sur tous les services
- Limites de ressources CPU et memoire (`deploy.resources`)
- Profil Spring `production` active
- Ports de bases de donnees non exposes (`ports: []`)
- Mot de passe Redis obligatoire
- Options JVM optimisees

---

## docker-compose.monitoring.yml (monitoring stack)

```yaml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus-data:/prometheus
    ports:
      - "${PROMETHEUS_PORT:-9090}:9090"
    networks:
      - backend
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=30d"

  grafana:
    image: grafana/grafana
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "${GRAFANA_PORT:-3000}:3000"
    depends_on:
      - prometheus
    networks:
      - backend

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - backend

  kibana:
    image: docker.elastic.co/kibana/kibana:8.12.0
    container_name: kibana
    environment:
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - backend

  zipkin:
    image: openzipkin/zipkin
    container_name: zipkin
    ports:
      - "${ZIPKIN_PORT:-9411}:9411"
    environment:
      STORAGE_TYPE: elasticsearch
      ES_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    networks:
      - backend

volumes:
  prometheus-data:
  grafana-data:
  elasticsearch-data:
```

---

## Variables d'environnement (.env)

Reference complete de toutes les variables d'environnement :

### General

| Variable               | Description                   | Defaut       |
| ---------------------- | ----------------------------- | ------------ |
| `COMPOSE_PROJECT_NAME` | Nom du projet Docker Compose  | `rofecare`   |
| `ENVIRONMENT`          | Environnement (dev/prod)      | `development`|

### Bases de donnees (par service)

Le modele est identique pour chaque service. Exemple avec Identity :

| Variable               | Description                | Defaut          |
| ---------------------- | -------------------------- | --------------- |
| `IDENTITY_DB_HOST`     | Hote PostgreSQL            | `identity-db`   |
| `IDENTITY_DB_PORT`     | Port PostgreSQL            | `5432`          |
| `IDENTITY_DB_NAME`     | Nom de la base             | `identity_db`   |
| `IDENTITY_DB_USER`     | Utilisateur PostgreSQL     | `identity_user` |
| `IDENTITY_DB_PASSWORD` | Mot de passe PostgreSQL    | `password`      |

Services concernes : `IDENTITY`, `PATIENT`, `CLINICAL`, `MEDTECH`, `PHARMACY`, `FINANCE`, `PLATFORM`, `INTEROP`.

### Redis

| Variable         | Description       | Defaut     |
| ---------------- | ----------------- | ---------- |
| `REDIS_HOST`     | Hote Redis        | `redis`    |
| `REDIS_PORT`     | Port Redis        | `6379`     |
| `REDIS_PASSWORD` | Mot de passe      | `password` |

### Kafka

| Variable                   | Description              | Defaut          |
| -------------------------- | ------------------------ | --------------- |
| `KAFKA_BOOTSTRAP_SERVERS`  | Serveurs Kafka           | `kafka:29092`   |
| `ZOOKEEPER_HOST`           | Hote Zookeeper           | `zookeeper`     |
| `ZOOKEEPER_PORT`           | Port Zookeeper           | `2181`          |

### Spring Cloud

| Variable                   | Description                       | Defaut                |
| -------------------------- | --------------------------------- | --------------------- |
| `CONFIG_SERVER_PORT`       | Port du Config Server             | `8888`                |
| `CONFIG_SERVER_GIT_URI`    | Repository Git de configuration   | -                     |
| `DISCOVERY_SERVER_PORT`    | Port du Discovery Server          | `8761`                |
| `DISCOVERY_SERVER_HOST`    | Hote du Discovery Server          | `discovery-server`    |
| `GATEWAY_PORT`             | Port du API Gateway               | `8080`                |

### Securite

| Variable                 | Description              | Defaut   |
| ------------------------ | ------------------------ | -------- |
| `JWT_SECRET`             | Cle secrete JWT          | -        |
| `JWT_EXPIRATION`         | Expiration JWT (ms)      | `86400000` |
| `JWT_REFRESH_EXPIRATION` | Expiration refresh (ms)  | `604800000` |

### Monitoring

| Variable                 | Description                    | Defaut  |
| ------------------------ | ------------------------------ | ------- |
| `PROMETHEUS_PORT`        | Port Prometheus                | `9090`  |
| `GRAFANA_PORT`           | Port Grafana                   | `3000`  |
| `GRAFANA_ADMIN_PASSWORD` | Mot de passe admin Grafana     | `admin` |
| `ZIPKIN_PORT`            | Port Zipkin                    | `9411`  |

---

## Images Docker utilisees

### Services applicatifs

| Image                            | Usage                  | Taille approximative |
| -------------------------------- | ---------------------- | -------------------- |
| `eclipse-temurin:21-jre-alpine`  | Runtime des services   | ~180 MB              |
| `eclipse-temurin:21-jdk-alpine`  | Build stage (multi-stage) | ~350 MB           |

### Bases de donnees et cache

| Image                                | Usage                     | Version |
| ------------------------------------ | ------------------------- | ------- |
| `postgres:16-alpine`                 | Bases de donnees          | 16.x    |
| `redis:7-alpine`                     | Cache et sessions         | 7.x     |

### Messagerie

| Image                                | Usage                     | Version |
| ------------------------------------ | ------------------------- | ------- |
| `confluentinc/cp-zookeeper:7.6.0`   | Coordination Kafka        | 7.6.0   |
| `confluentinc/cp-kafka:7.6.0`       | Bus de messages           | 7.6.0   |
| `provectuslabs/kafka-ui`            | UI d'administration Kafka | latest  |

### Monitoring

| Image                                                        | Usage               | Version |
| ------------------------------------------------------------ | ------------------- | ------- |
| `prom/prometheus`                                            | Collecte de metriques | latest |
| `grafana/grafana`                                            | Dashboards           | latest  |
| `docker.elastic.co/elasticsearch/elasticsearch:8.12.0`       | Moteur de recherche  | 8.12.0  |
| `docker.elastic.co/kibana/kibana:8.12.0`                     | UI Elasticsearch     | 8.12.0  |
| `openzipkin/zipkin`                                          | Tracing distribue    | latest  |

---

## Dockerfile multi-stage build

Chaque service utilise un Dockerfile multi-stage pour optimiser la taille de l'image finale :

```dockerfile
# ============================================================
# Stage 1 : Build
# ============================================================
FROM eclipse-temurin:21-jdk-alpine AS build

WORKDIR /app

# Copier les fichiers Maven (cache des dependances)
COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn

# Telecharger les dependances (couche en cache)
RUN ./mvnw dependency:go-offline -B

# Copier le code source
COPY src ./src

# Compiler l'application
RUN ./mvnw clean package -DskipTests -B

# ============================================================
# Stage 2 : Runtime
# ============================================================
FROM eclipse-temurin:21-jre-alpine

# Creer un utilisateur non-root
RUN addgroup -S rofecare && adduser -S rofecare -G rofecare

WORKDIR /app

# Copier le JAR depuis le stage de build
COPY --from=build /app/target/*.jar app.jar

# Changer le proprietaire
RUN chown -R rofecare:rofecare /app

# Utiliser l'utilisateur non-root
USER rofecare

# Exposer le port du service
EXPOSE 8081

# Point d'entree
ENTRYPOINT ["java", \
  "-XX:+UseG1GC", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### Avantages du multi-stage build

| Aspect             | Description                                           |
| ------------------ | ----------------------------------------------------- |
| Taille de l'image  | Uniquement le JRE, pas le JDK (~180 MB vs ~350 MB)   |
| Securite           | Pas d'outils de build dans l'image finale             |
| Cache Docker       | Les dependances Maven sont mises en cache              |
| Utilisateur        | Execution en tant qu'utilisateur non-root             |

---

## Gestion des volumes

### Volumes nommes

```bash
# Lister les volumes
docker volume ls | grep rofecare

# Inspecter un volume
docker volume inspect rofecare_identity-db-data

# Sauvegarder un volume
docker run --rm \
  -v rofecare_identity-db-data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/identity-db-data.tar.gz -C /data .

# Restaurer un volume
docker run --rm \
  -v rofecare_identity-db-data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/identity-db-data.tar.gz -C /data
```

### Liste des volumes

| Volume                 | Service           | Contenu                     |
| ---------------------- | ----------------- | --------------------------- |
| `identity-db-data`     | identity-db       | Donnees PostgreSQL          |
| `patient-db-data`      | patient-db        | Donnees PostgreSQL          |
| `clinical-db-data`     | clinical-db       | Donnees PostgreSQL          |
| `medtech-db-data`      | medtech-db        | Donnees PostgreSQL          |
| `pharmacy-db-data`     | pharmacy-db       | Donnees PostgreSQL          |
| `finance-db-data`      | finance-db        | Donnees PostgreSQL          |
| `platform-db-data`     | platform-db       | Donnees PostgreSQL          |
| `interop-db-data`      | interop-db        | Donnees PostgreSQL          |
| `redis-data`           | redis             | Donnees de cache            |
| `zookeeper-data`       | zookeeper         | Donnees Zookeeper           |
| `zookeeper-logs`       | zookeeper         | Logs Zookeeper              |
| `kafka-data`           | kafka             | Messages Kafka              |
| `prometheus-data`      | prometheus        | Metriques (monitoring)      |
| `grafana-data`         | grafana           | Dashboards (monitoring)     |
| `elasticsearch-data`   | elasticsearch     | Index ELK (monitoring)      |

---

## Configuration reseau

### Reseaux Docker

| Reseau     | Type   | Description                                      |
| ---------- | ------ | ------------------------------------------------ |
| `frontend` | bridge | Communication externe (Gateway <-> clients)      |
| `backend`  | bridge | Communication inter-services                     |
| `database` | bridge | Communication services <-> bases de donnees      |

### Isolation reseau

```
Internet
    |
[frontend] ---- gateway
    |               |
    |          [backend] ---- services metier ---- redis, kafka
    |               |
    |          [database] ---- PostgreSQL instances
```

En production, les reseaux `backend` et `database` peuvent etre configures comme `internal: true` pour empecher tout acces Internet direct.

---

## Commandes Docker courantes

### Gestion du cycle de vie

```bash
# Demarrer tous les services
docker compose up -d

# Arreter tous les services
docker compose down

# Arreter et supprimer les volumes (ATTENTION : perte de donnees)
docker compose down -v

# Redemarrer un service specifique
docker compose restart patient-service

# Reconstruire et redemarrer un service
docker compose up -d --build patient-service
```

### Supervision

```bash
# Voir l'etat de tous les services
docker compose ps

# Voir les logs (temps reel)
docker compose logs -f

# Voir les logs d'un service (derniere heure)
docker compose logs --since 1h patient-service

# Voir l'utilisation des ressources
docker stats --no-stream

# Inspecter un conteneur
docker inspect patient-service
```

### Maintenance

```bash
# Executer une commande dans un conteneur
docker compose exec patient-db psql -U patient_user patient_db

# Copier des fichiers depuis un conteneur
docker compose cp patient-service:/app/logs ./patient-logs

# Nettoyer les ressources inutilisees
docker system prune -f

# Nettoyer les images non utilisees
docker image prune -a -f

# Mettre a jour les images
docker compose pull
docker compose up -d
```

### Scaling

```bash
# Scaler un service
docker compose up -d --scale patient-service=3

# Verifier les instances
docker compose ps patient-service
```
