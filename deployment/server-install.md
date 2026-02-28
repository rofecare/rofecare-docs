# Installation serveur (microservices)

Guide de deploiement de Rofecare HIS en architecture microservices sur un **serveur distant** (VPS, cloud, machine dediee). Pour l'installation locale sur un ordinateur de bureau ou portable, voir [local-install.md](local-install.md).

---

## Table des matieres

- [Prerequisites](#prerequisites)
- [Telecharger les images Docker](#telecharger-les-images-docker)
- [Configuration de l'environnement](#configuration-de-lenvironnement)
- [Demarrage avec Docker Compose (4 phases)](#demarrage-avec-docker-compose-4-phases)
- [Scripts de demarrage](#scripts-de-demarrage)
- [Verification du deploiement](#verification-du-deploiement)
- [Securisation](#securisation)
- [Ressources et optimisation](#ressources-et-optimisation)
- [Monitoring et observabilite](#monitoring-et-observabilite)
- [Sauvegardes](#sauvegardes)
- [Haute disponibilite et scaling](#haute-disponibilite-et-scaling)
- [Gestion des logs](#gestion-des-logs)
- [Reference des ports](#reference-des-ports)

---

## Prerequisites

### Materiel

| Ressource | Auto-heberge (tout inclus) | Optimise (DB managee) |
| --------- | -------------------------- | --------------------- |
| RAM       | 16-22 GB                   | **8 GB**              |
| CPU       | 8 coeurs                   | 4 coeurs              |
| Disque    | 200 GB SSD                 | 50 GB SSD             |
| Reseau    | 100 Mbps                   | 100 Mbps              |

> **Profil optimise** : En utilisant un PostgreSQL manage (DigitalOcean, Supabase, Neon) et Kafka KRaft (sans Zookeeper), la RAM necessaire passe de ~22 GB a **~8 GB**. Voir [Ressources et optimisation](#ressources-et-optimisation) pour les details.

### Logiciels

| Composant       | Version minimale |
| --------------- | ---------------- |
| Docker Engine   | 24.x            |
| Docker Compose  | 2.20+            |
| Git             | 2.x              |

> **Note** : Java et Maven ne sont pas necessaires pour le deploiement. Les images Docker pre-compilees sont telecharges directement depuis GitHub Container Registry (GHCR).

### Verification

```bash
docker --version          # Docker version 24.x+
docker compose version    # Docker Compose version v2.20+
git --version             # git version 2.x
```

---

## Telecharger les images Docker

Les images Docker sont compilees et publiees automatiquement par le pipeline CI/CD a chaque push sur la branche `main`, sur **GitHub Container Registry (GHCR)**.

### Authentification GHCR

```bash
# Creez un Personal Access Token sur https://github.com/settings/tokens
# Scope necessaire : read:packages
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
```

### Telecharger toutes les images

```bash
for service in \
  rofecare-service-identity \
  rofecare-service-patient \
  rofecare-service-clinical \
  rofecare-service-medical-technology \
  rofecare-service-pharmacy \
  rofecare-service-finance \
  rofecare-service-platform \
  rofecare-service-interoperability \
  rofecare-config-server \
  rofecare-discovery-server \
  rofecare-gateway; do
  echo "Pulling $service..."
  docker pull ghcr.io/rofecare/$service:latest
done
```

### Telecharger une version specifique

```bash
docker pull ghcr.io/rofecare/rofecare-service-identity:v1.0.0   # par tag de version
docker pull ghcr.io/rofecare/rofecare-service-identity:a1b2c3d   # par SHA du commit
```

### Verifier les images

```bash
docker images | grep ghcr.io/rofecare
```

---

## Configuration de l'environnement

```bash
# Cloner uniquement l'infrastructure (docker-compose + scripts)
mkdir -p /opt/rofecare && cd /opt/rofecare
git clone https://github.com/rofecare/rofecare-infrastructure.git
cd rofecare-infrastructure/docker
cp .env.example .env
```

Editez le fichier `.env` avec les valeurs appropriees :

```env
# ============================================================
# GENERAL
# ============================================================
COMPOSE_PROJECT_NAME=rofecare
ENVIRONMENT=production

# ============================================================
# DATABASES - Un PostgreSQL par service
# ============================================================
# Identity Service
IDENTITY_DB_HOST=identity-db
IDENTITY_DB_PORT=5432
IDENTITY_DB_NAME=identity_db
IDENTITY_DB_USER=identity_user
IDENTITY_DB_PASSWORD=changez_moi_identity

# Patient Service
PATIENT_DB_HOST=patient-db
PATIENT_DB_PORT=5432
PATIENT_DB_NAME=patient_db
PATIENT_DB_USER=patient_user
PATIENT_DB_PASSWORD=changez_moi_patient

# Clinical Service
CLINICAL_DB_HOST=clinical-db
CLINICAL_DB_PORT=5432
CLINICAL_DB_NAME=clinical_db
CLINICAL_DB_USER=clinical_user
CLINICAL_DB_PASSWORD=changez_moi_clinical

# Medical Technology Service
MEDTECH_DB_HOST=medtech-db
MEDTECH_DB_PORT=5432
MEDTECH_DB_NAME=medtech_db
MEDTECH_DB_USER=medtech_user
MEDTECH_DB_PASSWORD=changez_moi_medtech

# Pharmacy Service
PHARMACY_DB_HOST=pharmacy-db
PHARMACY_DB_PORT=5432
PHARMACY_DB_NAME=pharmacy_db
PHARMACY_DB_USER=pharmacy_user
PHARMACY_DB_PASSWORD=changez_moi_pharmacy

# Finance Service
FINANCE_DB_HOST=finance-db
FINANCE_DB_PORT=5432
FINANCE_DB_NAME=finance_db
FINANCE_DB_USER=finance_user
FINANCE_DB_PASSWORD=changez_moi_finance

# Platform Service
PLATFORM_DB_HOST=platform-db
PLATFORM_DB_PORT=5432
PLATFORM_DB_NAME=platform_db
PLATFORM_DB_USER=platform_user
PLATFORM_DB_PASSWORD=changez_moi_platform

# Interoperability Service
INTEROP_DB_HOST=interop-db
INTEROP_DB_PORT=5432
INTEROP_DB_NAME=interop_db
INTEROP_DB_USER=interop_user
INTEROP_DB_PASSWORD=changez_moi_interop

# ============================================================
# REDIS
# ============================================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=changez_moi_redis

# ============================================================
# KAFKA
# ============================================================
KAFKA_BOOTSTRAP_SERVERS=kafka:29092
ZOOKEEPER_HOST=zookeeper
ZOOKEEPER_PORT=2181

# ============================================================
# SPRING CLOUD
# ============================================================
CONFIG_SERVER_PORT=8888
CONFIG_SERVER_GIT_URI=https://github.com/rofecare/rofecare-config.git

DISCOVERY_SERVER_PORT=8761
DISCOVERY_SERVER_HOST=discovery-server

GATEWAY_PORT=8080

# ============================================================
# SECURITE
# ============================================================
JWT_SECRET=votre_cle_jwt_secrete_minimum_256_bits_production
JWT_EXPIRATION=86400000
JWT_REFRESH_EXPIRATION=604800000

# ============================================================
# MONITORING (optionnel)
# ============================================================
PROMETHEUS_PORT=9090
GRAFANA_PORT=3000
GRAFANA_ADMIN_PASSWORD=changez_moi_grafana
ZIPKIN_PORT=9411
```

---

## Demarrage avec Docker Compose (3 phases)

Le demarrage se fait en 3 phases distinctes pour respecter les dependances entre les services.

### Phase 1 : Infrastructure

Demarrez les services d'infrastructure (bases de donnees, cache, messagerie) :

```bash
cd /opt/rofecare/rofecare-infrastructure/docker

# Demarrer l'infrastructure
docker compose up -d \
  identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka

# Attendre que tous les services soient sains
echo "Attente du demarrage de l'infrastructure..."
sleep 30

# Verifier l'etat des bases de donnees
docker compose ps | grep -E "(db|redis|kafka|zookeeper)"
```

**Services demarres :**

| Service           | Description                  | Port expose |
| ----------------- | ---------------------------- | ----------- |
| `identity-db`     | PostgreSQL - Identite        | 5433        |
| `patient-db`      | PostgreSQL - Patients        | 5434        |
| `clinical-db`     | PostgreSQL - Clinique        | 5435        |
| `medtech-db`      | PostgreSQL - Technologie med.| 5436        |
| `pharmacy-db`     | PostgreSQL - Pharmacie       | 5437        |
| `finance-db`      | PostgreSQL - Finance         | 5438        |
| `platform-db`     | PostgreSQL - Plateforme      | 5439        |
| `interop-db`      | PostgreSQL - Interoperabilite| 5440        |
| `redis`           | Cache Redis                  | 6379        |
| `zookeeper`       | Coordination Kafka           | 2181        |
| `kafka`           | Bus de messages              | 9092        |

### Phase 2 : Spring Cloud

Demarrez les services Spring Cloud dans l'ordre strict suivant :

```bash
# 1. Config Server - centralise la configuration
docker compose up -d config-server
echo "Attente du Config Server (30s)..."
sleep 30

# Verifier que le Config Server est pret
curl -s http://localhost:8888/actuator/health | grep -q '"UP"' && \
  echo "Config Server OK" || echo "Config Server pas encore pret"

# 2. Discovery Server (Eureka) - registre des services
docker compose up -d discovery-server
echo "Attente du Discovery Server (30s)..."
sleep 30

# Verifier que Eureka est pret
curl -s http://localhost:8761/actuator/health | grep -q '"UP"' && \
  echo "Discovery Server OK" || echo "Discovery Server pas encore pret"

# 3. API Gateway - point d'entree unique
docker compose up -d gateway
echo "Attente du Gateway (20s)..."
sleep 20

curl -s http://localhost:8080/actuator/health | grep -q '"UP"' && \
  echo "Gateway OK" || echo "Gateway pas encore pret"
```

### Phase 3 : Services metier

Une fois l'infrastructure et Spring Cloud operationnels, demarrez tous les services metier en parallele :

```bash
# Demarrer tous les services metier simultanement
docker compose up -d \
  identity-service \
  patient-service \
  clinical-service \
  medical-technology-service \
  pharmacy-service \
  finance-service \
  platform-service \
  interoperability-service

echo "Attente du demarrage des services metier (60s)..."
sleep 60

# Verifier l'etat de tous les services
docker compose ps
```

### Phase 4 : Caddy (reverse proxy + TLS)

Demarrez Caddy pour exposer les services via les sous-domaines `*.rofecare.com` avec HTTPS automatique :

```bash
# Demarrer Caddy
docker compose up -d caddy

# Verifier que Caddy est pret
curl -s https://api.rofecare.com/actuator/health
```

> **Note** : Caddy obtient automatiquement les certificats TLS via Let's Encrypt. Assurez-vous que les DNS `*.rofecare.com` pointent vers l'IP du serveur avant de demarrer Caddy.

**Sous-domaines actifs apres cette phase :**

| Sous-domaine | Service |
|---|---|
| `api.rofecare.com` | API Gateway → tous les services |
| `app.rofecare.com` | Frontend Nuxt |
| `eureka.rofecare.com` | Dashboard Eureka (protege) |
| `config.rofecare.com` | Config Server (protege) |
| `grafana.rofecare.com` | Dashboards monitoring (protege) |
| `kibana.rofecare.com` | Logs centralises (protege) |
| `zipkin.rofecare.com` | Tracing distribue (protege) |
| `kafka-ui.rofecare.com` | Topics Kafka (protege) |
| `prometheus.rofecare.com` | Metriques (protege) |

---

## Scripts de demarrage

### `start-all.sh` - Demarrage complet

```bash
#!/bin/bash
# start-all.sh - Demarre l'ensemble de la plateforme Rofecare
set -e

COMPOSE_DIR="/opt/rofecare/rofecare-infrastructure/docker"
cd "$COMPOSE_DIR"

echo "=== Phase 1 : Infrastructure ==="
docker compose up -d \
  identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka

echo "Attente de l'infrastructure (30s)..."
sleep 30

echo "=== Phase 2 : Spring Cloud ==="
docker compose up -d config-server
echo "Attente du Config Server (30s)..."
sleep 30

docker compose up -d discovery-server
echo "Attente du Discovery Server (30s)..."
sleep 30

docker compose up -d gateway
echo "Attente du Gateway (20s)..."
sleep 20

echo "=== Phase 3 : Services metier ==="
docker compose up -d \
  identity-service patient-service clinical-service \
  medical-technology-service pharmacy-service \
  finance-service platform-service interoperability-service

echo "Attente des services (60s)..."
sleep 60

echo "=== Phase 4 : Caddy (reverse proxy + TLS) ==="
docker compose up -d caddy
echo "Attente de Caddy (10s)..."
sleep 10

echo "=== Verification ==="
docker compose ps
echo ""
echo "Application      : https://app.rofecare.com"
echo "API Gateway      : https://api.rofecare.com"
echo "Dashboard Eureka : https://eureka.rofecare.com"
echo "Grafana          : https://grafana.rofecare.com"
echo "Kafka UI         : https://kafka-ui.rofecare.com"
```

### `start-infra.sh` - Infrastructure uniquement

```bash
#!/bin/bash
# start-infra.sh - Demarre uniquement l'infrastructure
set -e

COMPOSE_DIR="/opt/rofecare/rofecare-infrastructure/docker"
cd "$COMPOSE_DIR"

echo "Demarrage de l'infrastructure..."
docker compose up -d \
  identity-db patient-db clinical-db medtech-db \
  pharmacy-db finance-db platform-db interop-db \
  redis zookeeper kafka

echo "Attente du demarrage (30s)..."
sleep 30

echo "Verification..."
docker compose ps | grep -E "(db|redis|kafka|zookeeper)"

echo ""
echo "Infrastructure prete. Vous pouvez demarrer les services dans votre IDE."
```

Rendez les scripts executables :

```bash
chmod +x start-all.sh start-infra.sh
```

---

## Verification du deploiement

### Health checks

```bash
# Verifier chaque service individuellement
services=(
  "config-server:8888"
  "discovery-server:8761"
  "gateway:8080"
  "identity-service:8081"
  "patient-service:8082"
  "clinical-service:8083"
  "medical-technology-service:8084"
  "pharmacy-service:8085"
  "finance-service:8086"
  "platform-service:8087"
  "interoperability-service:8088"
)

for svc in "${services[@]}"; do
  name="${svc%%:*}"
  port="${svc##*:}"
  status=$(curl -s -o /dev/null -w "%{http_code}" "http://localhost:$port/actuator/health" 2>/dev/null)
  if [ "$status" = "200" ]; then
    echo "[OK]   $name (port $port)"
  else
    echo "[FAIL] $name (port $port) - HTTP $status"
  fi
done
```

### Dashboard Eureka

Accedez au dashboard Eureka pour verifier que tous les services sont enregistres :

```
https://eureka.rofecare.com
```

Tous les services metier doivent apparaitre avec le statut **UP**.

### Tester l'API Gateway

```bash
# Verifier que le routage fonctionne via le gateway
curl https://api.rofecare.com/api/identity/actuator/health
curl https://api.rofecare.com/api/patients/actuator/health
curl https://api.rofecare.com/api/clinical/actuator/health
```

---

## Reference des ports

### Services Spring Cloud

| Service           | Port interne | Sous-domaine                    |
| ----------------- | ------------ | ------------------------------- |
| Config Server     | 8888         | https://config.rofecare.com     |
| Discovery Server  | 8761         | https://eureka.rofecare.com     |
| API Gateway       | 8080         | https://api.rofecare.com        |

### Services metier (accessibles via API Gateway)

| Service                     | Port interne | Route via api.rofecare.com          |
| --------------------------- | ------------ | ----------------------------------- |
| Identity Service            | 8081         | `https://api.rofecare.com/api/identity/**` |
| Patient Service             | 8082         | `https://api.rofecare.com/api/patients/**` |
| Clinical Service            | 8083         | `https://api.rofecare.com/api/clinical/**` |
| Medical Technology Service  | 8084         | `https://api.rofecare.com/api/medical-technology/**` |
| Pharmacy Service            | 8085         | `https://api.rofecare.com/api/pharmacy/**` |
| Finance Service             | 8086         | `https://api.rofecare.com/api/finance/**` |
| Platform Service            | 8087         | `https://api.rofecare.com/api/platform/**` |
| Interoperability Service    | 8088         | `https://api.rofecare.com/api/interoperability/**` |

> **Note** : Les ports internes (8081-8088) ne sont pas exposes en production. Tous les appels passent par `api.rofecare.com` (Caddy → API Gateway → Service).

### Infrastructure

| Service      | Port interne | Sous-domaine / Acces              |
| ------------ | ------------ | --------------------------------- |
| PostgreSQL   | 5432         | Interne uniquement (pas expose)   |
| Redis        | 6379         | Interne uniquement (pas expose)   |
| Kafka        | 9092         | Interne uniquement (pas expose)   |
| Kafka UI     | 8180         | https://kafka-ui.rofecare.com     |
| Prometheus   | 9090         | https://prometheus.rofecare.com   |
| Grafana      | 3001         | https://grafana.rofecare.com      |
| Kibana       | 5601         | https://kibana.rofecare.com       |
| Zipkin       | 9411         | https://zipkin.rofecare.com       |

---

## Securisation

### Generer des mots de passe securises

> **Critique** : ne jamais utiliser les mots de passe par defaut en production.

```bash
openssl rand -base64 32  # Pour chaque base de donnees
openssl rand -base64 64  # Pour le JWT secret
openssl rand -base64 32  # Pour Redis
```

Mettez a jour toutes les variables dans le fichier `.env` :

```env
IDENTITY_DB_PASSWORD=<mot_de_passe_genere_1>
PATIENT_DB_PASSWORD=<mot_de_passe_genere_2>
CLINICAL_DB_PASSWORD=<mot_de_passe_genere_3>
# ... pour chaque service

REDIS_PASSWORD=<mot_de_passe_genere_redis>
JWT_SECRET=<cle_jwt_generee_64_caracteres>
CONFIG_ENCRYPT_KEY=<cle_chiffrement_config>
GRAFANA_ADMIN_PASSWORD=<mot_de_passe_grafana>
```

### Configuration JWT production

```yaml
# application-production.yml
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 3600000        # 1 heure
    refresh-expiration: 86400000  # 24 heures
    issuer: rofecare-production
    audience: rofecare-api
```

### Caddy — Reverse proxy et TLS automatique

Caddy gere automatiquement les certificats TLS via Let's Encrypt (HTTPS, renouvellement, HTTP/3).

```caddyfile
# /etc/caddy/Caddyfile

# API Gateway
api.rofecare.com {
    reverse_proxy localhost:8080

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Frame-Options DENY
        X-Content-Type-Options nosniff
        X-XSS-Protection "1; mode=block"
    }
}

# Application frontend
app.rofecare.com {
    reverse_proxy localhost:3000
}

# ============================================================
# Infrastructure & Monitoring (protege par basicauth)
# ============================================================
# Generer le hash : caddy hash-password --plaintext 'VotreMotDePasse'

(infra_auth) {
    basicauth {
        {$CADDY_ADMIN_USER:admin} {$CADDY_ADMIN_PASSWORD_HASH}
    }
}

grafana.rofecare.com {
    import infra_auth
    reverse_proxy localhost:3001
}

kibana.rofecare.com {
    import infra_auth
    reverse_proxy localhost:5601
}

eureka.rofecare.com {
    import infra_auth
    reverse_proxy localhost:8761
}

zipkin.rofecare.com {
    import infra_auth
    reverse_proxy localhost:9411
}

kafka-ui.rofecare.com {
    import infra_auth
    reverse_proxy localhost:8180
}

prometheus.rofecare.com {
    import infra_auth
    reverse_proxy localhost:9090
}

config.rofecare.com {
    import infra_auth
    reverse_proxy localhost:8888
}
```

#### Configurer l'authentification infrastructure

```bash
# Generer le hash bcrypt
caddy hash-password --plaintext 'VotreMotDePasseSecurise'

# Ajouter dans .env
CADDY_ADMIN_USER=admin
CADDY_ADMIN_PASSWORD_HASH=$2a$14$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

#### Docker Compose pour Caddy

```yaml
caddy:
  image: caddy:2-alpine
  restart: unless-stopped
  ports:
    - "80:80"
    - "443:443"
    - "443:443/udp"  # HTTP/3
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile:ro
    - caddy_data:/data
    - caddy_config:/config
  networks:
    - frontend

volumes:
  caddy_data:
  caddy_config:
```

### Securite reseau Docker

```yaml
# docker-compose.prod.yml - Reseaux isoles
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Pas d'acces Internet
  database:
    driver: bridge
    internal: true
```

### Docker Compose production overlay

```bash
# Demarrage en mode production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

```yaml
# docker-compose.prod.yml (extrait)
services:
  identity-db:
    restart: always
    ports: []  # Ne pas exposer les ports DB en production

  redis:
    restart: always
    ports: []
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
```

---

## Ressources et optimisation

### Limites par type de service

| Type de service          | CPU | RAM (limite) | RAM (reserve) |
| ------------------------ | --- | ------------ | ------------- |
| Spring Cloud (Config, Discovery) | 0.5 | 512 MB  | 256 MB        |
| API Gateway              | 1.0 | 768 MB       | 384 MB        |
| Services metier          | 1.0 | 512 MB       | 256 MB        |
| Clinical Service         | 1.0 | 768 MB       | 384 MB        |
| Redis                    | 0.5 | 256 MB       | 128 MB        |
| Kafka KRaft              | 1.0 | 1 GB         | 512 MB        |

### JVM tuning

```yaml
environment:
  JAVA_OPTS: "-Xms128m -Xmx384m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+UseStringDeduplication"
```

### Profil complet — auto-heberge (~22 GB)

| Composants                      | RAM estimee |
| ------------------------------- | ----------- |
| 8 bases PostgreSQL              | ~6 GB       |
| 8 services metier               | ~9 GB       |
| 3 services Spring Cloud         | ~2 GB       |
| Redis + Kafka + Zookeeper       | ~3 GB       |
| Monitoring                      | ~2 GB       |
| **Total**                       | **~22 GB**  |

### Profil optimise — DB managee (~8 GB, recommande)

| Composants                             | RAM estimee |
| -------------------------------------- | ----------- |
| PostgreSQL manage (distant)            | **0 GB**    |
| 8 services metier (JVM optimisee)      | ~4 GB       |
| 3 services Spring Cloud               | ~1.5 GB     |
| Redis                                  | ~256 MB     |
| Kafka KRaft (sans Zookeeper)           | ~1 GB       |
| Monitoring leger                       | ~1 GB       |
| Caddy                                  | ~50 MB      |
| **Total**                              | **~8 GB**   |

### Configuration DB managee (DigitalOcean)

```env
DB_HOST=db-postgresql-rofecare-do-user-xxxxx.ondigitalocean.com
DB_PORT=25060
DB_SSL_MODE=require

IDENTITY_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_identity?sslmode=require
PATIENT_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_patient?sslmode=require
CLINICAL_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_clinical?sslmode=require
MEDTECH_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_medical_technology?sslmode=require
PHARMACY_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_pharmacy?sslmode=require
FINANCE_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_finance?sslmode=require
PLATFORM_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_platform?sslmode=require
INTEROP_DB_URL=jdbc:postgresql://${DB_HOST}:${DB_PORT}/rofecare_interoperability?sslmode=require
```

### Kafka KRaft (sans Zookeeper)

```yaml
kafka:
  image: confluentinc/cp-kafka:7.6.0
  environment:
    KAFKA_NODE_ID: 1
    KAFKA_PROCESS_ROLES: broker,controller
    KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
    KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
    KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
    KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    CLUSTER_ID: "rofecare-kafka-cluster-001"
  deploy:
    resources:
      limits:
        memory: 1G
```

### Providers PostgreSQL manage

| Provider | Plan | Prix | Avantages |
|---|---|---|---|
| **DigitalOcean** | Managed DB Basic | ~15 $/mois | Simple, backups auto |
| **Supabase** | Pro | ~25 $/mois | Postgres + API REST |
| **Neon** | Scale | ~19 $/mois | Serverless, scale-to-zero |
| **AWS RDS** | db.t4g.micro | ~15 $/mois | Multi-AZ disponible |

### PostgreSQL tuning

```sql
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '768MB';
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET maintenance_work_mem = '128MB';
ALTER SYSTEM SET max_connections = 100;
SELECT pg_reload_conf();
```

### Connection pooling (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1200000
      connection-timeout: 30000
```

---

## Monitoring et observabilite

### Demarrer le stack monitoring

```bash
docker compose -f docker-compose.yml \
  -f docker-compose.prod.yml \
  -f docker-compose.monitoring.yml up -d
```

### Prometheus

Fichier `prometheus/prometheus.yml` :

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "spring-boot-services"
    metrics_path: /actuator/prometheus
    scrape_interval: 10s
    static_configs:
      - targets:
          - gateway:8080
          - identity-service:8081
          - patient-service:8082
          - clinical-service:8083
          - medical-technology-service:8084
          - pharmacy-service:8085
          - finance-service:8086
          - platform-service:8087
          - interoperability-service:8088

  - job_name: "postgresql"
    static_configs:
      - targets: [postgres-exporter:9187]

  - job_name: "redis"
    static_configs:
      - targets: [redis-exporter:9121]

  - job_name: "kafka"
    static_configs:
      - targets: [kafka-exporter:9308]
```

### Alertes Prometheus

```yaml
# prometheus/alert-rules.yml
groups:
  - name: rofecare-alerts
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} indisponible"

      - alert: HighMemoryUsage
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
        for: 5m
        labels:
          severity: warning

      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
```

### Grafana

Accedez a `https://grafana.rofecare.com` (utilisateur : `admin`).

| Dashboard | ID Grafana | Description |
|---|---|---|
| Spring Boot Statistics | 12464 | Metriques JVM et Spring Boot |
| PostgreSQL Database | 9628 | Surveillance PostgreSQL |
| Redis Dashboard | 11835 | Metriques Redis |
| Kafka Overview | 7589 | Supervision Kafka |
| Docker Containers | 893 | Ressources conteneurs |

### ELK (Elasticsearch + Kibana)

```yaml
# docker-compose.monitoring.yml (extrait)
elasticsearch:
  image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
  environment:
    - discovery.type=single-node
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data

kibana:
  image: docker.elastic.co/kibana/kibana:8.12.0
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
  depends_on:
    - elasticsearch
```

### Zipkin (tracing distribue)

Accedez a `https://zipkin.rofecare.com`.

```yaml
# application-production.yml
management:
  tracing:
    sampling:
      probability: 0.1  # 10% des requetes en production
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

### Endpoints Actuator

| Endpoint | Description |
|---|---|
| `/actuator/health` | Etat de sante global |
| `/actuator/health/liveness` | Sonde de vivacite (Kubernetes) |
| `/actuator/health/readiness` | Sonde de disponibilite |
| `/actuator/metrics` | Metriques Micrometer |
| `/actuator/prometheus` | Metriques format Prometheus |

### Script de monitoring

```bash
#!/bin/bash
# health-check.sh
SERVICES=(
  "API Gateway:https://api.rofecare.com/actuator/health"
  "Identity:https://api.rofecare.com/api/identity/actuator/health"
  "Patient:https://api.rofecare.com/api/patients/actuator/health"
  "Clinical:https://api.rofecare.com/api/clinical/actuator/health"
  "Med Tech:https://api.rofecare.com/api/medical-technology/actuator/health"
  "Pharmacy:https://api.rofecare.com/api/pharmacy/actuator/health"
  "Finance:https://api.rofecare.com/api/finance/actuator/health"
  "Platform:https://api.rofecare.com/api/platform/actuator/health"
  "Interop:https://api.rofecare.com/api/interoperability/actuator/health"
  "Eureka:https://eureka.rofecare.com/actuator/health"
  "Grafana:https://grafana.rofecare.com/api/health"
)

echo "=== Rofecare Health Check - $(date) ==="
ALL_OK=true

for entry in "${SERVICES[@]}"; do
  url="${entry#*:}"
  name="${entry%%:*}"
  response=$(curl -s -o /dev/null -w "%{http_code}" "$url" --max-time 5 2>/dev/null)

  if [ "$response" = "200" ]; then
    echo "[UP]   $name"
  else
    echo "[DOWN] $name (HTTP $response)"
    ALL_OK=false
  fi
done

if [ "$ALL_OK" = false ]; then
  echo ""
  echo "ALERTE : un ou plusieurs services sont indisponibles."
  exit 1
fi

echo "Tous les services sont operationnels."
```

---

## Sauvegardes

### Script de sauvegarde quotidienne

```bash
#!/bin/bash
# backup-databases.sh
set -e

BACKUP_DIR="/opt/rofecare/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

databases=(
  "identity-db:identity_db:identity_user"
  "patient-db:patient_db:patient_user"
  "clinical-db:clinical_db:clinical_user"
  "medtech-db:medtech_db:medtech_user"
  "pharmacy-db:pharmacy_db:pharmacy_user"
  "finance-db:finance_db:finance_user"
  "platform-db:platform_db:platform_user"
  "interop-db:interop_db:interop_user"
)

echo "=== Sauvegarde Rofecare - $DATE ==="

for entry in "${databases[@]}"; do
  IFS=":" read -r container dbname dbuser <<< "$entry"
  BACKUP_FILE="$BACKUP_DIR/${dbname}_${DATE}.sql.gz"

  echo "Sauvegarde de $dbname..."
  docker exec "$container" pg_dump -U "$dbuser" "$dbname" | gzip > "$BACKUP_FILE"

  SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
  echo "  OK - $BACKUP_FILE ($SIZE)"
done

echo "Nettoyage des sauvegardes de plus de $RETENTION_DAYS jours..."
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "=== Sauvegarde terminee ==="
```

### Planification cron

```bash
# Sauvegarde quotidienne a 2h du matin
echo "0 2 * * * /opt/rofecare/scripts/backup-databases.sh >> /var/log/rofecare/backup.log 2>&1" | crontab -
```

### Restauration

```bash
gunzip -c /opt/rofecare/backups/patient_db_20260228_020000.sql.gz | \
  docker exec -i patient-db psql -U patient_user patient_db
```

---

## Haute disponibilite et scaling

### Scaling horizontal

```bash
# Scaler les services critiques
docker compose up -d --scale patient-service=3 --scale clinical-service=2

# Verifier les instances
docker compose ps patient-service
```

Eureka et Spring Cloud LoadBalancer gerent automatiquement la decouverte et la repartition de charge.

### Recommandations HA

| Composant | Strategie |
|---|---|
| PostgreSQL | Streaming replication + Patroni |
| Redis | Redis Sentinel ou Redis Cluster |
| Kafka | Cluster multi-broker (min. 3) |
| Services metier | Instances multiples via Gateway |
| Load Balancer | Caddy (reverse proxy + TLS auto) |

### Arreter la plateforme

```bash
# Arret gracieux
docker compose down

# Arret avec suppression des volumes (ATTENTION : perte de donnees)
docker compose down -v
```

---

## Gestion des logs

### Configuration logback production

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="production">
        <appender name="JSON_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>/var/log/rofecare/${spring.application.name}.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>/var/log/rofecare/${spring.application.name}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>100MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>5GB</totalSizeCap>
            </rollingPolicy>
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>

        <root level="WARN">
            <appender-ref ref="JSON_FILE"/>
        </root>
        <logger name="com.rofecare" level="INFO"/>
    </springProfile>
</configuration>
```

### Rotation et retention

| Parametre | Valeur | Description |
|---|---|---|
| Taille max | 100 MB | Par fichier de log |
| Historique | 30 jours | Conservation des archives |
| Taille totale | 5 GB | Limite par service |
| Format | JSON | Pour ingestion ELK |

### Consulter les logs Docker

```bash
docker compose logs -f patient-service --tail 100   # Un service
docker compose logs -f --tail 50                      # Tous les services
docker compose logs --no-color > rofecare-logs-$(date +%Y%m%d).txt  # Exporter
```
