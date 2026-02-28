# Installation serveur (microservices)

Guide de deploiement de Rofecare HIS en architecture microservices, concu pour les environnements serveur avec Docker et Docker Compose.

---

## Table des matieres

- [Prerequisites](#prerequisites)
- [Cloner les repositories](#cloner-les-repositories)
- [Configuration de l'environnement](#configuration-de-lenvironnement)
- [Compilation des services](#compilation-des-services)
- [Demarrage avec Docker Compose (3 phases)](#demarrage-avec-docker-compose-3-phases)
- [Scripts de demarrage](#scripts-de-demarrage)
- [Verification du deploiement](#verification-du-deploiement)
- [Reference des ports](#reference-des-ports)
- [Mise a l'echelle des services](#mise-a-lechelle-des-services)

---

## Prerequisites

### Materiel

| Ressource | Minimum   | Recommande |
| --------- | --------- | ---------- |
| RAM       | 16 GB     | 32 GB      |
| CPU       | 4 coeurs  | 8 coeurs   |
| Disque    | 50 GB SSD | 200 GB SSD |
| Reseau    | 100 Mbps  | 1 Gbps     |

### Logiciels

| Composant       | Version minimale |
| --------------- | ---------------- |
| Docker Engine   | 24.x            |
| Docker Compose  | 2.20+            |
| Git             | 2.x              |
| Java 21 (build) | Eclipse Temurin 21 |
| Maven           | 3.9+             |

### Verification

```bash
docker --version          # Docker version 24.x+
docker compose version    # Docker Compose version v2.20+
git --version             # git version 2.x
java -version             # openjdk version "21.x.x"
mvn -version              # Apache Maven 3.9.x
```

---

## Cloner les repositories

```bash
# Creer le repertoire de travail
mkdir -p /opt/rofecare && cd /opt/rofecare

# Cloner le backend principal
git clone https://github.com/your-org/rofecare-server.git

# Cloner le frontend (optionnel, si deploiement integre)
git clone https://github.com/your-org/rofecare-frontend.git
```

---

## Configuration de l'environnement

Creez un fichier `.env` a la racine du projet `rofecare-server` :

```bash
cd /opt/rofecare/rofecare-server
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
CONFIG_SERVER_GIT_URI=https://github.com/your-org/rofecare-config.git

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

## Compilation des services

### Etape 1 : Compiler `rofecare-common`

Le module commun doit etre compile et installe dans le repository Maven local en premier :

```bash
cd /opt/rofecare/rofecare-server/rofecare-common
mvn clean install -DskipTests
```

### Etape 2 : Compiler tous les services

```bash
cd /opt/rofecare/rofecare-server

# Compiler tous les modules
mvn clean package -DskipTests

# Ou compiler service par service
for service in rofecare-identity-service rofecare-patient-service \
               rofecare-clinical-service rofecare-medical-technology-service \
               rofecare-pharmacy-service rofecare-finance-service \
               rofecare-platform-service rofecare-interoperability-service; do
  echo "Building $service..."
  cd /opt/rofecare/rofecare-server/$service
  mvn clean package -DskipTests
done
```

### Etape 3 : Construire les images Docker

```bash
cd /opt/rofecare/rofecare-server
docker compose build
```

---

## Demarrage avec Docker Compose (3 phases)

Le demarrage se fait en 3 phases distinctes pour respecter les dependances entre les services.

### Phase 1 : Infrastructure

Demarrez les services d'infrastructure (bases de donnees, cache, messagerie) :

```bash
cd /opt/rofecare/rofecare-server

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

---

## Scripts de demarrage

### `start-all.sh` - Demarrage complet

```bash
#!/bin/bash
# start-all.sh - Demarre l'ensemble de la plateforme Rofecare
set -e

COMPOSE_DIR="/opt/rofecare/rofecare-server"
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

echo "=== Verification ==="
docker compose ps
echo ""
echo "Dashboard Eureka : http://localhost:8761"
echo "API Gateway      : http://localhost:8080"
echo "Kafka UI         : http://localhost:9000"
```

### `start-infra.sh` - Infrastructure uniquement

```bash
#!/bin/bash
# start-infra.sh - Demarre uniquement l'infrastructure
set -e

COMPOSE_DIR="/opt/rofecare/rofecare-server"
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
http://localhost:8761
```

Tous les services metier doivent apparaitre avec le statut **UP**.

### Tester l'API Gateway

```bash
# Verifier que le routage fonctionne via le gateway
curl http://localhost:8080/api/v1/identity/actuator/health
curl http://localhost:8080/api/v1/patients/actuator/health
curl http://localhost:8080/api/v1/clinical/actuator/health
```

---

## Reference des ports

### Services Spring Cloud

| Service           | Port interne | Port expose | URL                           |
| ----------------- | ------------ | ----------- | ----------------------------- |
| Config Server     | 8888         | 8888        | http://localhost:8888          |
| Discovery Server  | 8761         | 8761        | http://localhost:8761          |
| API Gateway       | 8080         | 8080        | http://localhost:8080          |

### Services metier

| Service                     | Port interne | Port expose |
| --------------------------- | ------------ | ----------- |
| Identity Service            | 8081         | 8081        |
| Patient Service             | 8082         | 8082        |
| Clinical Service            | 8083         | 8083        |
| Medical Technology Service  | 8084         | 8084        |
| Pharmacy Service            | 8085         | 8085        |
| Finance Service             | 8086         | 8086        |
| Platform Service            | 8087         | 8087        |
| Interoperability Service    | 8088         | 8088        |

### Infrastructure

| Service      | Port interne | Port expose | Usage                    |
| ------------ | ------------ | ----------- | ------------------------ |
| PostgreSQL   | 5432         | 5433-5440   | Bases de donnees         |
| Redis        | 6379         | 6379        | Cache                    |
| Zookeeper    | 2181         | 2181        | Coordination Kafka       |
| Kafka        | 29092        | 9092        | Bus de messages          |
| Kafka UI     | 8080         | 9000        | Interface administration |

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
