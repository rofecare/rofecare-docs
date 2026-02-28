# Installation serveur (microservices)

Guide de deploiement de Rofecare HIS en architecture microservices sur un **serveur distant** (VPS, cloud, machine dediee). Pour l'installation locale sur un ordinateur de bureau ou portable, voir [local-install.md](local-install.md).

---

## Table des matieres

- [Prerequisites](#prerequisites)
- [Telecharger les images Docker](#telecharger-les-images-docker)
- [Configuration de l'environnement](#configuration-de-lenvironnement)

- [Demarrage avec Docker Compose (3 phases)](#demarrage-avec-docker-compose-3-phases)
- [Scripts de demarrage](#scripts-de-demarrage)
- [Verification du deploiement](#verification-du-deploiement)
- [Reference des ports](#reference-des-ports)
- [Mise a l'echelle des services](#mise-a-lechelle-des-services)

---

## Prerequisites

### Materiel

| Ressource | Auto-heberge (tout inclus) | Optimise (DB managee) |
| --------- | -------------------------- | --------------------- |
| RAM       | 16-22 GB                   | **8 GB**              |
| CPU       | 8 coeurs                   | 4 coeurs              |
| Disque    | 200 GB SSD                 | 50 GB SSD             |
| Reseau    | 100 Mbps                   | 100 Mbps              |

> **Profil optimise** : En utilisant un PostgreSQL manage (DigitalOcean, Supabase, Neon) et Kafka KRaft (sans Zookeeper), la RAM necessaire passe de ~22 GB a **~8 GB**. Voir [production.md](../deployment/production.md#profils-de-deploiement) pour les details.

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

## Mise a l'echelle des services

### Augmenter le nombre d'instances

Docker Compose permet de scaler horizontalement les services metier :

```bash
# Scaler un service specifique
docker compose up -d --scale patient-service=3
docker compose up -d --scale clinical-service=2

# Verifier les instances
docker compose ps patient-service
```

### Considerations pour le scaling

- **Eureka** : les instances supplementaires s'enregistrent automatiquement
- **Gateway** : le load balancing est gere par Spring Cloud LoadBalancer
- **Kafka** : chaque instance rejoint le consumer group automatiquement
- **Base de donnees** : partagee entre les instances d'un meme service (attention aux connexions)

### Limiter les ressources par service

```bash
# Voir l'utilisation des ressources
docker stats --no-stream

# Les limites sont definies dans docker-compose.yml via deploy.resources
```

### Arreter la plateforme

```bash
# Arret gracieux de tous les services
docker compose down

# Arret avec suppression des volumes (ATTENTION : perte de donnees)
docker compose down -v
```
