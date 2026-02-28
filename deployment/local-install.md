# Installation locale en milieu hospitalier

Guide d'installation de Rofecare HIS dans un environnement hospitalier local, en mode monolithe ou via l'application desktop Tauri.

---

## Table des matieres

- [Prerequisites](#prerequisites)
- [Mode monolithe (JAR unique)](#mode-monolithe-jar-unique)
- [Configuration de la base de donnees](#configuration-de-la-base-de-donnees)
- [Configuration applicative](#configuration-applicative)
- [Demarrage de l'application](#demarrage-de-lapplication)
- [Application desktop (Tauri)](#application-desktop-tauri)
- [Synchronisation avec le cloud](#synchronisation-avec-le-cloud)
- [Fonctionnement hors ligne](#fonctionnement-hors-ligne)
- [Depannage](#depannage)

---

## Prerequisites

### Obligatoires

| Composant    | Version minimale | Recommandee      |
| ------------ | ---------------- | ----------------- |
| Java (JRE)   | 21               | Eclipse Temurin 21 |
| PostgreSQL   | 16               | 16.x              |
| RAM          | 4 GB             | 8 GB               |
| Disque       | 10 GB            | 50 GB+             |

### Optionnels

| Composant       | Usage                                      |
| --------------- | ------------------------------------------ |
| Docker + Compose | Deploiement conteneurise simplifie        |
| Redis 7         | Cache et gestion de sessions (recommande)  |

### Verification des prerequisites

```bash
# Verifier Java
java -version
# Attendu : openjdk version "21.x.x"

# Verifier PostgreSQL
psql --version
# Attendu : psql (PostgreSQL) 16.x

# Verifier Docker (optionnel)
docker --version
docker compose version
```

---

## Mode monolithe (JAR unique)

Le mode monolithe regroupe tous les modules metier dans un seul artefact executable. Ce mode est ideal pour les installations locales avec des ressources limitees.

### Obtenir le JAR

Telechargez la derniere version depuis la page [GitHub Releases](https://github.com/your-org/rofecare-server/releases) :

```bash
# Telecharger la derniere release
wget https://github.com/your-org/rofecare-server/releases/latest/download/rofecare-server.jar
```

Ou compilez depuis les sources :

```bash
git clone https://github.com/your-org/rofecare-server.git
cd rofecare-server

# Compiler le module commun d'abord
cd rofecare-common
mvn clean install -DskipTests

# Compiler le serveur monolithe
cd ../rofecare-server
mvn clean package -DskipTests -Pmonolith
```

---

## Configuration de la base de donnees

### Creer la base de donnees

```bash
# Se connecter a PostgreSQL
sudo -u postgres psql

# Creer l'utilisateur et la base
CREATE USER rofecare WITH PASSWORD 'votre_mot_de_passe_securise';
CREATE DATABASE rofecare_db OWNER rofecare;
GRANT ALL PRIVILEGES ON DATABASE rofecare_db TO rofecare;

# Activer l'extension UUID
\c rofecare_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

\q
```

### Avec Docker (alternative)

```bash
docker run -d \
  --name rofecare-postgres \
  -e POSTGRES_USER=rofecare \
  -e POSTGRES_PASSWORD=votre_mot_de_passe_securise \
  -e POSTGRES_DB=rofecare_db \
  -p 5432:5432 \
  -v rofecare-pgdata:/var/lib/postgresql/data \
  postgres:16-alpine
```

---

## Configuration applicative

Creez un fichier `application-monolith.yml` dans le meme repertoire que le JAR :

```yaml
spring:
  profiles:
    active: monolith

  datasource:
    url: jdbc:postgresql://localhost:5432/rofecare_db
    username: rofecare
    password: votre_mot_de_passe_securise
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true

  # Redis (optionnel - desactiver si non disponible)
  data:
    redis:
      host: localhost
      port: 6379
      enabled: false  # Mettre a true si Redis est installe

# Configuration du serveur
server:
  port: 8080

# Configuration JWT
app:
  jwt:
    secret: votre_cle_jwt_secrete_minimum_256_bits
    expiration: 86400000  # 24 heures en millisecondes
    refresh-expiration: 604800000  # 7 jours

# Configuration monolithe - desactive les composants distribues
rofecare:
  monolith:
    enabled: true
    kafka:
      enabled: false
    eureka:
      enabled: false
```

### Variables d'environnement (alternative)

Vous pouvez egalement configurer via des variables d'environnement :

```bash
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/rofecare_db
export SPRING_DATASOURCE_USERNAME=rofecare
export SPRING_DATASOURCE_PASSWORD=votre_mot_de_passe_securise
export APP_JWT_SECRET=votre_cle_jwt_secrete_minimum_256_bits
```

---

## Demarrage de l'application

### Demarrage simple

```bash
java -jar rofecare-server.jar --spring.profiles.active=monolith
```

### Demarrage avec configuration externe

```bash
java -jar rofecare-server.jar \
  --spring.profiles.active=monolith \
  --spring.config.additional-location=file:./application-monolith.yml
```

### Demarrage avec options JVM optimisees

```bash
java \
  -Xms512m \
  -Xmx2g \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -jar rofecare-server.jar \
  --spring.profiles.active=monolith
```

### Verifier le demarrage

```bash
# Attendre le demarrage complet, puis verifier
curl http://localhost:8080/actuator/health
# Attendu : {"status":"UP"}
```

L'interface web est accessible a l'adresse : `http://localhost:8080`

### Installation en tant que service systemd (Linux)

Creez le fichier `/etc/systemd/system/rofecare.service` :

```ini
[Unit]
Description=Rofecare HIS Server
After=postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=rofecare
WorkingDirectory=/opt/rofecare
ExecStart=/usr/bin/java -Xms512m -Xmx2g -jar /opt/rofecare/rofecare-server.jar --spring.profiles.active=monolith
SuccessExitStatus=143
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable rofecare
sudo systemctl start rofecare
sudo systemctl status rofecare
```

---

## Application desktop (Tauri)

L'application desktop Rofecare utilise **Tauri 2.x** pour offrir une experience native sur toutes les plateformes.

### Architecture Tauri

```
+--------------------------------------------------+
|              Application Tauri                    |
|                                                   |
|  +---------------------------------------------+ |
|  |           Shell Rust (Tauri Core)            | |
|  |  - Gestion du cycle de vie                   | |
|  |  - Communication IPC                         | |
|  |  - Acces systeme de fichiers                 | |
|  +---------------------------------------------+ |
|                                                   |
|  +---------------------------------------------+ |
|  |         WebView2 (Frontend Nuxt 3)           | |
|  |  - Interface utilisateur                     | |
|  |  - Rendu HTML/CSS/JS                         | |
|  +---------------------------------------------+ |
|                                                   |
|  +---------------------------------------------+ |
|  |           Sidecars embarques                 | |
|  |  - PostgreSQL (base locale)                  | |
|  |  - JRE 21 + rofecare-server.jar              | |
|  +---------------------------------------------+ |
+--------------------------------------------------+
```

- **Shell Rust** : coquille native performante et securisee gerant les interactions systeme
- **WebView2** : moteur de rendu web natif (Edge/WebKit selon la plateforme)
- **Sidecars** : PostgreSQL et le JRE sont embarques, aucune installation externe requise

### Installateurs disponibles

| Plateforme | Format           | Fichier                           |
| ---------- | ---------------- | --------------------------------- |
| macOS      | `.dmg`           | `Rofecare_x.y.z_aarch64.dmg`     |
| macOS      | `.dmg`           | `Rofecare_x.y.z_x64.dmg`         |
| Windows    | `.msi`           | `Rofecare_x.y.z_x64_en-US.msi`   |
| Linux      | `.AppImage`      | `Rofecare_x.y.z_amd64.AppImage`  |
| Linux      | `.deb`           | `rofecare_x.y.z_amd64.deb`       |

### Installation par plateforme

#### macOS

```bash
# Telecharger le .dmg depuis la page Releases
# Double-cliquer sur le fichier .dmg
# Glisser Rofecare dans le dossier Applications

# Si macOS bloque l'application (Gatekeeper)
xattr -cr /Applications/Rofecare.app
```

#### Windows

```
1. Telecharger le fichier .msi
2. Double-cliquer pour lancer l'installateur
3. Suivre l'assistant d'installation
4. Rofecare sera disponible dans le menu Demarrer
```

#### Linux

```bash
# Option 1 : AppImage
chmod +x Rofecare_x.y.z_amd64.AppImage
./Rofecare_x.y.z_amd64.AppImage

# Option 2 : Paquet .deb (Debian/Ubuntu)
sudo dpkg -i rofecare_x.y.z_amd64.deb
sudo apt-get install -f  # Resoudre les dependances si necessaire
```

### Premier demarrage

Au premier lancement, l'application desktop :

1. Initialise la base de donnees PostgreSQL locale embarquee
2. Demarre le serveur backend en mode monolithe
3. Ouvre l'interface utilisateur dans la WebView
4. Propose la creation du compte administrateur initial

---

## Synchronisation avec le cloud

L'installation locale peut se synchroniser avec une instance cloud de Rofecare pour centraliser les donnees.

### Configurer la synchronisation

Dans l'application, accedez a **Parametres > Synchronisation** ou ajoutez au fichier de configuration :

```yaml
rofecare:
  sync:
    enabled: true
    cloud-url: https://cloud.rofecare.example.com
    facility-id: HOSP-001
    api-key: votre_cle_api_synchronisation
    interval: 300  # Intervalle en secondes (5 minutes)
    batch-size: 100
    conflict-resolution: cloud-wins  # ou local-wins
```

### Modes de synchronisation

| Mode            | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `bidirectional` | Synchronisation dans les deux sens (par defaut)              |
| `push-only`     | Envoie les donnees locales vers le cloud uniquement          |
| `pull-only`     | Recoit les donnees du cloud uniquement                       |

### Premiere synchronisation

```bash
# Declencher une synchronisation initiale complete
curl -X POST http://localhost:8080/api/v1/sync/full \
  -H "Authorization: Bearer <token>"
```

---

## Fonctionnement hors ligne

Rofecare est concu pour fonctionner de maniere autonome meme sans connexion reseau.

### Capacites hors ligne

- Toutes les fonctionnalites cliniques restent disponibles
- Les donnees sont stockees localement dans PostgreSQL
- Les fichiers et images sont sauvegardes sur le disque local
- Une file d'attente de synchronisation accumule les modifications

### Reprise de connexion

Lorsque la connexion est retablie :

1. Les modifications en attente sont synchronisees automatiquement
2. Les conflits sont resolus selon la strategie configuree
3. Un rapport de synchronisation est genere
4. Les alertes et notifications du cloud sont recuperees

### Indicateurs de statut

L'interface affiche clairement le statut de connexion :

- **Vert** : connecte et synchronise
- **Orange** : connecte, synchronisation en cours
- **Rouge** : hors ligne, modifications en file d'attente

---

## Depannage

### L'application ne demarre pas

```bash
# Verifier que le port 8080 n'est pas deja utilise
lsof -i :8080  # macOS/Linux
netstat -ano | findstr :8080  # Windows

# Verifier les logs
tail -f /var/log/rofecare/application.log
# ou dans le repertoire courant
tail -f ./logs/spring.log
```

### Erreur de connexion a la base de donnees

```bash
# Verifier que PostgreSQL est en cours d'execution
sudo systemctl status postgresql  # Linux
brew services list  # macOS

# Tester la connexion
psql -h localhost -U rofecare -d rofecare_db -c "SELECT 1;"
```

### Problemes de memoire

```bash
# Augmenter la memoire allouee a la JVM
java -Xms1g -Xmx4g -jar rofecare-server.jar --spring.profiles.active=monolith

# Verifier l'utilisation memoire
curl http://localhost:8080/actuator/metrics/jvm.memory.used
```

### L'application desktop ne se lance pas

| Probleme                          | Solution                                         |
| --------------------------------- | ------------------------------------------------ |
| Ecran blanc                       | Verifier les logs dans `~/.rofecare/logs/`       |
| Erreur WebView2 (Windows)         | Installer Microsoft Edge WebView2 Runtime        |
| Permission refusee (Linux)        | `chmod +x` sur l'AppImage                        |
| Gatekeeper bloque (macOS)         | `xattr -cr /Applications/Rofecare.app`           |
| Base de donnees corrompue         | Supprimer `~/.rofecare/data/` et relancer        |

### Problemes de synchronisation

```bash
# Verifier la connectivite avec le serveur cloud
curl -I https://cloud.rofecare.example.com/actuator/health

# Forcer une resynchronisation
curl -X POST http://localhost:8080/api/v1/sync/force \
  -H "Authorization: Bearer <token>"

# Consulter la file d'attente de synchronisation
curl http://localhost:8080/api/v1/sync/queue/status \
  -H "Authorization: Bearer <token>"
```

### Reinitialisation complete

> **Attention** : cette operation supprime toutes les donnees locales.

```bash
# Arreter l'application
sudo systemctl stop rofecare

# Supprimer et recreer la base de donnees
sudo -u postgres psql -c "DROP DATABASE rofecare_db;"
sudo -u postgres psql -c "CREATE DATABASE rofecare_db OWNER rofecare;"

# Redemarrer
sudo systemctl start rofecare
```
