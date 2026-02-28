# Deploiement en production

Guide complet pour le deploiement de Rofecare HIS dans un environnement de production securise, performant et supervise.

---

## Table des matieres

- [Docker Compose en production](#docker-compose-en-production)
- [Limites de ressources](#limites-de-ressources)
- [Securisation](#securisation)
- [Supervision et monitoring](#supervision-et-monitoring)
- [Strategie de sauvegarde](#strategie-de-sauvegarde)
- [Haute disponibilite](#haute-disponibilite)
- [Optimisation des performances](#optimisation-des-performances)
- [Gestion des logs](#gestion-des-logs)
- [Surveillance de la sante des services](#surveillance-de-la-sante-des-services)

---

## Docker Compose en production

Utilisez le fichier overlay `docker-compose.prod.yml` pour surcharger la configuration de developpement avec des parametres adaptes a la production :

```bash
# Demarrage en mode production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### Structure du fichier `docker-compose.prod.yml`

```yaml
version: "3.8"

services:
  # ---- Spring Cloud ----
  config-server:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - ENCRYPT_KEY=${CONFIG_ENCRYPT_KEY}

  discovery-server:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          memory: 256M

  gateway:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          memory: 512M
    environment:
      - SPRING_PROFILES_ACTIVE=production

  # ---- Services metier ----
  identity-service:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          memory: 512M
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - JAVA_OPTS=-Xms512m -Xmx768m

  patient-service:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
        reservations:
          memory: 512M
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - JAVA_OPTS=-Xms512m -Xmx768m

  clinical-service:
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 1.5G
        reservations:
          memory: 768M
    environment:
      - SPRING_PROFILES_ACTIVE=production
      - JAVA_OPTS=-Xms768m -Xmx1280m

  # ---- Bases de donnees ----
  identity-db:
    restart: always
    ports: []  # Ne pas exposer les ports en production
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  patient-db:
    restart: always
    ports: []
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G

  # (meme configuration pour les autres bases de donnees)

  # ---- Infrastructure ----
  redis:
    restart: always
    ports: []
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 256mb --maxmemory-policy allkeys-lru
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M

  kafka:
    restart: always
    ports: []
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 2G
```

---

## Limites de ressources

### Recommandations par type de service

| Type de service          | CPU (limites) | RAM (limites) | RAM (reserves) |
| ------------------------ | ------------- | ------------- | -------------- |
| Spring Cloud (Config, Discovery) | 0.5 CPU | 512 MB     | 256 MB         |
| API Gateway              | 1.0 CPU       | 1 GB          | 512 MB         |
| Services metier          | 1.0 CPU       | 1 GB          | 512 MB         |
| Clinical Service         | 1.5 CPU       | 1.5 GB        | 768 MB         |
| PostgreSQL (par instance)| 0.5-1.0 CPU   | 512 MB - 1 GB | 256 MB         |
| Redis                    | 0.5 CPU       | 512 MB        | 128 MB         |
| Kafka                    | 1.0 CPU       | 2 GB          | 1 GB           |
| Zookeeper                | 0.5 CPU       | 512 MB        | 256 MB         |

### Estimation totale

Pour un deploiement complet avec monitoring :

| Composants                      | RAM estimee |
| ------------------------------- | ----------- |
| 8 bases PostgreSQL              | ~6 GB       |
| 8 services metier               | ~9 GB       |
| 3 services Spring Cloud         | ~2 GB       |
| Redis + Kafka + Zookeeper       | ~3 GB       |
| Monitoring (Prometheus, Grafana) | ~2 GB      |
| **Total**                       | **~22 GB**  |

---

## Securisation

### Changer les mots de passe par defaut

> **Critique** : ne jamais utiliser les mots de passe par defaut en production.

```bash
# Generer des mots de passe securises
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

### Desactiver les ports de base de donnees exposes

Dans `docker-compose.prod.yml`, retirez ou videz la section `ports` de chaque base de donnees :

```yaml
identity-db:
  ports: []  # Pas d'acces externe

patient-db:
  ports: []
```

Les services accedent aux bases de donnees via le reseau Docker interne.

### Configurer les secrets JWT

```yaml
# application-production.yml
app:
  jwt:
    secret: ${JWT_SECRET}  # Minimum 256 bits, genere avec openssl
    expiration: 3600000     # 1 heure en production (plus court)
    refresh-expiration: 86400000  # 24 heures
    issuer: rofecare-production
    audience: rofecare-api
```

### Configuration TLS/HTTPS

#### Reverse proxy Caddy (recommande)

Caddy gere automatiquement les certificats TLS via Let's Encrypt (HTTPS automatique, renouvellement inclus).

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

# Monitoring (acces restreint)
grafana.rofecare.com {
    reverse_proxy localhost:3001
}

kibana.rofecare.com {
    reverse_proxy localhost:5601
}

eureka.rofecare.com {
    reverse_proxy localhost:8761
}

zipkin.rofecare.com {
    reverse_proxy localhost:9411
}
```

> **Note** : Caddy obtient et renouvelle automatiquement les certificats TLS. Aucune configuration SSL manuelle n'est necessaire. La redirection HTTP → HTTPS est egalement automatique.

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

---

## Supervision et monitoring

### Prometheus + Grafana

#### Configuration Prometheus

Utilisez `docker-compose.monitoring.yml` pour demarrer le stack de monitoring :

```bash
docker compose -f docker-compose.yml \
  -f docker-compose.prod.yml \
  -f docker-compose.monitoring.yml up -d
```

Fichier `prometheus/prometheus.yml` :

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

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
      - targets:
          - postgres-exporter:9187

  - job_name: "redis"
    static_configs:
      - targets:
          - redis-exporter:9121

  - job_name: "kafka"
    static_configs:
      - targets:
          - kafka-exporter:9308
```

#### Dashboards Grafana

Accedez a Grafana sur `http://localhost:3000` (utilisateur : `admin`).

Dashboards recommandes a importer :

| Dashboard                    | ID Grafana | Description                          |
| ---------------------------- | ---------- | ------------------------------------ |
| Spring Boot Statistics       | 12464      | Metriques JVM et Spring Boot         |
| PostgreSQL Database          | 9628       | Surveillance PostgreSQL              |
| Redis Dashboard              | 11835      | Metriques Redis                      |
| Kafka Overview               | 7589       | Supervision Kafka                    |
| Docker Container Monitoring  | 893        | Ressources conteneurs                |

### Stack ELK (Elasticsearch, Logstash, Kibana)

Pour la centralisation des logs :

```yaml
# docker-compose.monitoring.yml (extrait)
elasticsearch:
  image: docker.elastic.co/elasticsearch/elasticsearch:8.12.0
  environment:
    - discovery.type=single-node
    - xpack.security.enabled=false
    - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
  volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
  ports:
    - "9200:9200"

kibana:
  image: docker.elastic.co/kibana/kibana:8.12.0
  environment:
    - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
  ports:
    - "5601:5601"
  depends_on:
    - elasticsearch
```

Configuration logback pour envoyer les logs a Logstash :

```xml
<!-- logback-spring.xml -->
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <destination>logstash:5044</destination>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"service":"${spring.application.name}"}</customFields>
    </encoder>
</appender>
```

### Zipkin (tracing distribue)

```yaml
# docker-compose.monitoring.yml (extrait)
zipkin:
  image: openzipkin/zipkin
  ports:
    - "9411:9411"
  environment:
    - STORAGE_TYPE=elasticsearch
    - ES_HOSTS=http://elasticsearch:9200
```

Configuration Spring Boot pour le tracing :

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

Accedez a l'interface Zipkin : `http://localhost:9411`

---

## Strategie de sauvegarde

### Sauvegarde PostgreSQL avec pg_dump

#### Script de sauvegarde quotidienne

```bash
#!/bin/bash
# backup-databases.sh
set -e

BACKUP_DIR="/opt/rofecare/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

mkdir -p "$BACKUP_DIR"

# Liste des bases de donnees
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

  if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    echo "  OK - $BACKUP_FILE ($SIZE)"
  else
    echo "  ERREUR - echec de la sauvegarde de $dbname"
  fi
done

# Nettoyage des anciennes sauvegardes
echo ""
echo "Nettoyage des sauvegardes de plus de $RETENTION_DAYS jours..."
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "=== Sauvegarde terminee ==="
```

#### Planification avec cron

```bash
# Sauvegarde quotidienne a 2h du matin
echo "0 2 * * * /opt/rofecare/scripts/backup-databases.sh >> /var/log/rofecare/backup.log 2>&1" | crontab -
```

#### Restauration

```bash
# Restaurer une base de donnees
gunzip -c /opt/rofecare/backups/patient_db_20260228_020000.sql.gz | \
  docker exec -i patient-db psql -U patient_user patient_db
```

---

## Haute disponibilite

### Recommandations

| Composant         | Strategie HA                                  |
| ----------------- | --------------------------------------------- |
| PostgreSQL        | Streaming replication + pgpool-II ou Patroni  |
| Redis             | Redis Sentinel ou Redis Cluster               |
| Kafka             | Cluster multi-broker (min. 3 brokers)         |
| Services metier   | Instances multiples derriere le Gateway       |
| API Gateway       | Instances multiples derriere un load balancer  |
| Load Balancer     | Caddy en amont (reverse proxy + TLS auto)     |

### PostgreSQL en haute disponibilite

```yaml
# Exemple avec PostgreSQL replication
primary-db:
  image: postgres:16-alpine
  environment:
    - POSTGRES_REPLICATION_MODE=master
    - POSTGRES_REPLICATION_USER=replicator
    - POSTGRES_REPLICATION_PASSWORD=${REPLICATION_PASSWORD}

replica-db:
  image: postgres:16-alpine
  environment:
    - POSTGRES_REPLICATION_MODE=slave
    - POSTGRES_MASTER_HOST=primary-db
    - POSTGRES_MASTER_PORT=5432
    - POSTGRES_REPLICATION_USER=replicator
    - POSTGRES_REPLICATION_PASSWORD=${REPLICATION_PASSWORD}
```

### Services multiples instances

```bash
# Scaler les services critiques
docker compose up -d --scale patient-service=3 --scale clinical-service=2
```

Eureka et Spring Cloud LoadBalancer gerent automatiquement la decouverte et la repartition de charge.

---

## Optimisation des performances

### JVM Tuning

```yaml
environment:
  - JAVA_OPTS=-Xms512m -Xmx768m
    -XX:+UseG1GC
    -XX:MaxGCPauseMillis=200
    -XX:+UseStringDeduplication
    -XX:+OptimizeStringConcat
    -Djava.security.egd=file:/dev/./urandom
```

### PostgreSQL Tuning

```sql
-- postgresql.conf (ajuster selon la RAM disponible)
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '768MB';
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM SET maintenance_work_mem = '128MB';
ALTER SYSTEM SET max_connections = 100;
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET random_page_cost = 1.1;  -- Pour SSD
SELECT pg_reload_conf();
```

### Connection pooling

Configurez HikariCP dans chaque service :

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000
      max-lifetime: 1200000
      connection-timeout: 30000
      leak-detection-threshold: 60000
```

### Redis caching

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1 heure
      cache-null-values: false
```

---

## Gestion des logs

### Configuration logback pour la production

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="production">
        <!-- Format JSON pour les logs -->
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

        <!-- Niveaux de log specifiques -->
        <logger name="com.rofecare" level="INFO"/>
        <logger name="org.springframework.security" level="WARN"/>
        <logger name="org.hibernate.SQL" level="WARN"/>
    </springProfile>
</configuration>
```

### Rotation et retention

| Parametre        | Valeur     | Description                         |
| ---------------- | ---------- | ----------------------------------- |
| Taille max       | 100 MB     | Par fichier de log                  |
| Historique       | 30 jours   | Conservation des archives           |
| Taille totale    | 5 GB       | Limite par service                  |
| Format           | JSON       | Pour ingestion ELK                  |

### Consulter les logs Docker

```bash
# Logs d'un service specifique
docker compose logs -f patient-service --tail 100

# Logs de tous les services
docker compose logs -f --tail 50

# Exporter les logs
docker compose logs --no-color > rofecare-logs-$(date +%Y%m%d).txt
```

---

## Surveillance de la sante des services

### Endpoints Actuator

Chaque service expose les endpoints de sante via Spring Boot Actuator :

| Endpoint                      | Description                      |
| ----------------------------- | -------------------------------- |
| `/actuator/health`            | Etat de sante global             |
| `/actuator/health/liveness`   | Sonde de vivacite (Kubernetes)   |
| `/actuator/health/readiness`  | Sonde de disponibilite           |
| `/actuator/info`              | Informations sur le service      |
| `/actuator/metrics`           | Metriques Micrometer             |
| `/actuator/prometheus`        | Metriques au format Prometheus   |

### Script de monitoring

```bash
#!/bin/bash
# health-check.sh - Verification periodique de la sante
SERVICES=(
  "API Gateway:http://localhost:8080/actuator/health"
  "Identity:http://localhost:8081/actuator/health"
  "Patient:http://localhost:8082/actuator/health"
  "Clinical:http://localhost:8083/actuator/health"
  "Med Tech:http://localhost:8084/actuator/health"
  "Pharmacy:http://localhost:8085/actuator/health"
  "Finance:http://localhost:8086/actuator/health"
  "Platform:http://localhost:8087/actuator/health"
  "Interop:http://localhost:8088/actuator/health"
)

echo "=== Rofecare Health Check - $(date) ==="
ALL_OK=true

for entry in "${SERVICES[@]}"; do
  IFS=":" read -r name url <<< "$entry"
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
  # Ajouter ici l'envoi de notification (email, Slack, etc.)
  exit 1
fi

echo ""
echo "Tous les services sont operationnels."
```

### Alertes Prometheus (alerting rules)

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
        annotations:
          summary: "Utilisation memoire elevee sur {{ $labels.application }}"

      - alert: HighResponseTime
        expr: http_server_requests_seconds_sum / http_server_requests_seconds_count > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Temps de reponse eleve sur {{ $labels.application }}"

      - alert: DatabaseConnectionPoolExhausted
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.9
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Pool de connexions presque epuise sur {{ $labels.application }}"
```
