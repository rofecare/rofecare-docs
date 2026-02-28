# Infrastructure

> **Module** : `rofecare-infrastructure/`
> **Role** : Configuration DevOps, Docker Compose et monitoring

## Vue d'ensemble

Le module infrastructure regroupe l'ensemble des fichiers de configuration necessaires au deploiement, a l'orchestration et a la supervision de la plateforme Rofecare. Il contient les definitions Docker Compose, les scripts de gestion du cycle de vie et la configuration du stack de monitoring.

## Structure du module

```
rofecare-infrastructure/
├── docker/
│   ├── docker-compose.yml              # Services principaux (bases de donnees, infra, 8 services metier)
│   ├── docker-compose.prod.yml         # Overlay de production (limites de ressources, ports DB non exposes)
│   ├── docker-compose.monitoring.yml   # Stack de monitoring (Prometheus, Grafana, ELK, Zipkin, Kafka UI)
│   └── .env                            # Variables d'environnement
├── scripts/
│   ├── start-all.sh                    # Demarrage complet (3 phases)
│   ├── start-infra.sh                  # Demarrage infrastructure seule
│   ├── build.sh                        # Construction des images Docker (11 services)
│   ├── stop-all.sh                     # Arret complet, suppression optionnelle des volumes
│   └── logs.sh <service>              # Consultation des logs en temps reel
└── monitoring/
    └── prometheus/                     # Configuration Prometheus
```

## Docker Compose

### docker-compose.yml — Services principaux

Ce fichier definit l'ensemble des conteneurs necessaires au fonctionnement de la plateforme.

#### Bases de donnees PostgreSQL

Chaque service metier dispose de sa propre instance PostgreSQL, assurant une isolation complete des donnees.

| Service | Port | Base de donnees | Health check |
|---|---|---|---|
| Identity Service | 5432 | `rofecare_identity` | `pg_isready` |
| Patient Service | 5433 | `rofecare_patient` | `pg_isready` |
| Clinical Service | 5434 | `rofecare_clinical` | `pg_isready` |
| Medical Technology Service | 5435 | `rofecare_medical_technology` | `pg_isready` |
| Pharmacy Service | 5436 | `rofecare_pharmacy` | `pg_isready` |
| Finance Service | 5437 | `rofecare_finance` | `pg_isready` |
| Platform Service | 5438 | `rofecare_platform` | `pg_isready` |
| Interoperability Service | 5439 | `rofecare_interoperability` | `pg_isready` |

Chaque instance utilise un health check base sur `pg_isready` pour signaler sa disponibilite aux services dependants.

#### Redis

| Composant | Port | Health check |
|---|---|---|
| Redis | 6379 | `redis-cli ping` |

Redis est utilise pour le cache distribue, les sessions et le rate limiting du gateway.

#### Kafka

| Composant | Port | Description |
|---|---|---|
| Zookeeper | 2181 | Coordination du cluster Kafka |
| Kafka | 9092 | Broker de messages |

Kafka assure la communication asynchrone entre les services (evenements domaine, notifications, synchronisation).

#### Services Spring Cloud

| Service | Port | Dependances |
|---|---|---|
| Config Server | 8888 | Aucune |
| Discovery Server | 8761 | Config Server |
| API Gateway | 8080 | Discovery Server, Config Server |

#### Services metier

Les 8 services metier (ports 8081-8088) dependent de :
- Leur base de donnees PostgreSQL respective.
- Le Config Server (configuration).
- Le Discovery Server (enregistrement).
- Redis et Kafka.

Tous les services exposent des health checks via Spring Boot Actuator (`/actuator/health`) avec des periodes de demarrage configurees entre 30 et 90 secondes selon la complexite du service.

### docker-compose.prod.yml — Overlay de production

Ce fichier s'utilise en complement du fichier principal :

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Differences par rapport au developpement :
- **Limites de ressources** : CPU et memoire plafonnes pour chaque conteneur.
- **Ports des bases de donnees** : non exposes a l'exterieur (accessibles uniquement via le reseau Docker interne).
- **Replicas** : possibilite de definir plusieurs instances par service.
- **Volumes persistants** : configuration pour la retention des donnees.

### docker-compose.monitoring.yml — Stack de monitoring

Ce fichier definit les outils de supervision, utilise independamment ou en complement :

```bash
docker compose -f docker-compose.monitoring.yml up -d
```

## Scripts

### start-all.sh — Demarrage complet

Demarre la plateforme en 3 phases sequentielles pour respecter les dependances :

| Phase | Composants | Description |
|---|---|---|
| Phase 1 | PostgreSQL (x8), Redis, Zookeeper, Kafka | Infrastructure de base |
| Phase 2 | Config Server, Discovery Server, Gateway | Services Spring Cloud |
| Phase 3 | 8 services metier | Services applicatifs |

Chaque phase attend la disponibilite des services (health checks) avant de passer a la suivante.

```bash
./scripts/start-all.sh
```

### start-infra.sh — Infrastructure seule

Demarre uniquement les composants d'infrastructure (bases de donnees, Redis, Kafka) sans les services applicatifs. Utile pour le developpement local lorsque les services sont lances depuis l'IDE.

```bash
./scripts/start-infra.sh
```

### build.sh — Construction des images

Construit les images Docker pour les 11 services (3 Spring Cloud + 8 metier).

```bash
./scripts/build.sh
```

### stop-all.sh — Arret complet

Arrete tous les conteneurs. L'option `-v` supprime egalement les volumes (donnees).

```bash
./scripts/stop-all.sh        # Arret sans suppression des volumes
./scripts/stop-all.sh -v     # Arret avec suppression des volumes
```

### logs.sh — Consultation des logs

Affiche les logs en temps reel d'un service specifique.

```bash
./scripts/logs.sh rofecare-service-identity
```

## Stack de monitoring

### Tableau des ports

| Outil | Port | Role |
|---|---|---|
| Prometheus | 9090 | Collecte de metriques |
| Grafana | 3001 | Tableaux de bord et visualisation |
| Elasticsearch | 9200 | Stockage des logs |
| Kibana | 5601 | Visualisation des logs |
| Zipkin | 9411 | Tracing distribue |
| Kafka UI | 8180 | Visualisation des topics Kafka |

### Prometheus

Collecte les metriques exposees par chaque service via les endpoints Actuator (`/actuator/metrics`). La configuration se trouve dans `monitoring/prometheus/`. Prometheus scrape chaque service a intervalles reguliers pour alimenter les tableaux de bord Grafana.

### Grafana

Fournit des tableaux de bord preconfigures pour visualiser les metriques des services : taux de requetes, latences, utilisation memoire, etat des circuit breakers, etc. Accessible sur le port `3001`.

### ELK (Elasticsearch + Kibana)

Elasticsearch (port `9200`) stocke les logs centralises de tous les services. Kibana (port `5601`) permet de rechercher, filtrer et visualiser ces logs via une interface web.

### Zipkin

Assure le tracing distribue des requetes a travers les microservices. Chaque requete recoit un identifiant de trace (`X-Trace-Id`) qui permet de suivre son parcours complet dans l'architecture. Accessible sur le port `9411`.

### Kafka UI

Interface web (port `8180`) pour visualiser les topics Kafka, les groupes de consommateurs, les offsets et les messages en transit.

## Tableau recapitulatif des ports

| Port | Composant |
|---|---|
| 2181 | Zookeeper |
| 3001 | Grafana |
| 5432-5439 | PostgreSQL (8 instances) |
| 5601 | Kibana |
| 6379 | Redis |
| 8080 | API Gateway |
| 8081-8088 | Services metier (8 services) |
| 8180 | Kafka UI |
| 8761 | Discovery Server (Eureka) |
| 8888 | Config Server |
| 9090 | Prometheus |
| 9092 | Kafka |
| 9200 | Elasticsearch |
| 9411 | Zipkin |

## Endpoints Actuator

Tous les services Spring Boot exposent les endpoints Actuator suivants :

| Endpoint | Description |
|---|---|
| `/actuator/health` | Etat de sante du service (utilise par Docker pour les health checks) |
| `/actuator/info` | Informations sur le service (version, description) |
| `/actuator/metrics` | Metriques Micrometer (scraped par Prometheus) |
| `/actuator/env` | Variables d'environnement et proprietes de configuration |
| `/actuator/refresh` | Rafraichissement de la configuration depuis le Config Server |

## Ordre de demarrage

L'ordre de demarrage est critique pour le bon fonctionnement de la plateforme :

```
PostgreSQL (x8) + Redis + Zookeeper + Kafka
         │
         ▼
    Config Server (8888)
         │
         ▼
   Discovery Server (8761)
         │
         ▼
     API Gateway (8080)
         │
         ▼
  Services metier (8081-8088)
```

Chaque couche doit etre operationnelle (health check OK) avant le demarrage de la couche suivante. Les periodes de demarrage varient de 30 secondes (infrastructure) a 90 secondes (services metier) pour laisser le temps aux applications Spring Boot de s'initialiser completement.
