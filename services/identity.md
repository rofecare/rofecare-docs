# Service Identity

## Vue d'ensemble

Le service **Identity** est le pilier central de la securite et de la gestion des utilisateurs dans Rofecare HIS. Il gere l'authentification, l'autorisation, la gestion des comptes utilisateurs, et l'emission de tokens JWT. Tous les autres microservices dependent de ce service pour valider les identites et les permissions des utilisateurs.

| Propriete | Valeur |
|---|---|
| Port | `8081` |
| Base de donnees | `rofecare_identity` (PostgreSQL, port `5432`) |
| Framework | Spring Boot 3.4.1, Java 21 |
| Architecture | Hexagonale / DDD |

## Architecture hexagonale

```
┌─────────────────────────────────────────────────────────────┐
│                    infrastructure/                           │
│  ┌───────────────────────┐  ┌────────────────────────────┐  │
│  │  adapter/input/rest   │  │  adapter/output/persistence│  │
│  │  - AuthController     │  │  - UserJpaRepository       │  │
│  │  - UserController     │  │  - RoleJpaRepository       │  │
│  │  - RoleController     │  │  - RefreshTokenRepository  │  │
│  └──────────┬────────────┘  └────────────┬───────────────┘  │
│             │                            │                  │
│  ┌──────────┴────────────────────────────┴───────────────┐  │
│  │                   config / security                   │  │
│  │  - JwtConfig, SecurityConfig, RedisConfig             │  │
│  └───────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    application/                              │
│  ┌──────────────┐ ┌──────────────┐ ┌─────────────────────┐  │
│  │    port/      │ │   usecase/   │ │   dto / mapper      │  │
│  │  - InputPort  │ │  - LoginUC   │ │  - UserDTO          │  │
│  │  - OutputPort │ │  - RegisterUC│ │  - RoleDTO          │  │
│  └──────────────┘ └──────────────┘ └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                      domain/                                 │
│  ┌──────────┐ ┌────────────┐ ┌───────────┐ ┌────────────┐  │
│  │  model    │ │ repository │ │  service   │ │  event     │  │
│  │  - User   │ │            │ │            │ │            │  │
│  │  - Role   │ │            │ │            │ │            │  │
│  │-Permission│ │            │ │            │ │            │  │
│  └──────────┘ └────────────┘ └───────────┘ └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Responsabilites

- **Gestion des utilisateurs** : CRUD complet, activation et desactivation des comptes
- **Authentification** : login, logout, rafraichissement de token
- **Autorisation** : gestion des roles, permissions, controle d'acces base sur les roles (RBAC)
- **Gestion des tokens JWT** : emission et validation des access tokens et refresh tokens
- **Politique de mot de passe** : minimum 8 caracteres, verrouillage apres 5 tentatives echouees
- **Mecanisme de verrouillage de compte** : protection contre les attaques par force brute
- **Gestion des sessions** : stockage et invalidation via Redis

## Modele de domaine

### User
Represente un utilisateur du systeme. Contient les informations d'identification, les credentials, le statut du compte (actif/inactif/verrouille), et les roles associes.

### Role
Definit un ensemble de permissions. Chaque utilisateur peut avoir un ou plusieurs roles (ex: `ADMIN`, `MEDECIN`, `INFIRMIER`, `PHARMACIEN`, `LABORANTIN`).

### Permission
Unite atomique d'autorisation. Chaque permission correspond a une action specifique sur une ressource (ex: `USER_CREATE`, `PATIENT_READ`, `PRESCRIPTION_WRITE`).

### RefreshToken
Token de longue duree utilise pour obtenir un nouveau access token sans re-authentification. Stocke en base de donnees avec une date d'expiration.

### LoginAttempt
Enregistre chaque tentative de connexion (reussie ou echouee) pour un utilisateur. Utilise pour le mecanisme de verrouillage apres 5 echecs consecutifs.

## Endpoints API

| Methode | Chemin | Description | Auth requise |
|---|---|---|---|
| `POST` | `/api/identity/authentication/login` | Authentifier un utilisateur et obtenir les tokens JWT | Non |
| `POST` | `/api/identity/authentication/logout` | Deconnecter l'utilisateur et invalider le token | Oui |
| `POST` | `/api/identity/authentication/refresh` | Rafraichir le access token avec un refresh token | Non (refresh token requis) |
| `GET` | `/api/identity/users` | Lister tous les utilisateurs (avec pagination) | Oui (`USER_READ`) |
| `POST` | `/api/identity/users` | Creer un nouvel utilisateur | Oui (`USER_CREATE`) |
| `GET` | `/api/identity/users/{id}` | Obtenir les details d'un utilisateur | Oui (`USER_READ`) |
| `PUT` | `/api/identity/users/{id}` | Modifier un utilisateur | Oui (`USER_UPDATE`) |
| `DELETE` | `/api/identity/users/{id}` | Desactiver un utilisateur | Oui (`USER_DELETE`) |
| `PUT` | `/api/identity/users/{id}/activate` | Activer un compte utilisateur | Oui (`USER_UPDATE`) |
| `PUT` | `/api/identity/users/{id}/deactivate` | Desactiver un compte utilisateur | Oui (`USER_UPDATE`) |
| `GET` | `/api/identity/roles` | Lister tous les roles | Oui (`ROLE_READ`) |
| `POST` | `/api/identity/roles` | Creer un nouveau role | Oui (`ROLE_CREATE`) |
| `GET` | `/api/identity/roles/{id}` | Obtenir les details d'un role | Oui (`ROLE_READ`) |
| `PUT` | `/api/identity/roles/{id}` | Modifier un role | Oui (`ROLE_UPDATE`) |
| `POST` | `/api/identity/roles/{id}/permissions` | Assigner des permissions a un role | Oui (`ROLE_UPDATE`) |
| `POST` | `/api/identity/users/{id}/roles` | Assigner des roles a un utilisateur | Oui (`USER_UPDATE`) |

## Evenements Kafka

### Evenements produits

| Topic | Evenement | Description | Payload |
|---|---|---|---|
| `user-events` | `user-created` | Emis lors de la creation d'un nouvel utilisateur | `userId`, `username`, `email`, `roles`, `timestamp` |
| `user-events` | `user-updated` | Emis lors de la modification d'un utilisateur | `userId`, `changedFields`, `timestamp` |
| `user-events` | `role-assigned` | Emis lorsqu'un role est assigne a un utilisateur | `userId`, `roleId`, `roleName`, `timestamp` |

### Evenements consommes

Aucun. Le service Identity est un producteur pur d'evenements.

## Securite et JWT

### Configuration JWT

| Parametre | Valeur |
|---|---|
| Algorithme | HMAC-SHA256 |
| Duree access token | 1 heure |
| Duree refresh token | 24 heures |
| Header | `Authorization: Bearer <token>` |
| Claim: roles | Liste des roles de l'utilisateur |
| Claim: permissions | Liste des permissions de l'utilisateur |

### Politique de mot de passe

- Longueur minimale : 8 caracteres
- Verrouillage du compte apres 5 tentatives echouees consecutives
- Les mots de passe sont hashes avec BCrypt

### Gestion des sessions

Le service utilise **Redis** pour le stockage des sessions et la liste noire des tokens invalides (blacklist). Lors d'un logout, le access token est ajoute a la blacklist Redis avec un TTL correspondant a son temps d'expiration restant.

## Dependencies externes

| Dependance | Type | Usage |
|---|---|---|
| PostgreSQL | Base de donnees | Stockage des utilisateurs, roles, permissions, tokens |
| Redis | Cache / Session | Gestion des sessions, blacklist de tokens |
| Kafka | Messaging | Publication des evenements utilisateur |

Ce service ne depend d'aucun autre microservice Rofecare (pas de Feign clients). Il est consomme par tous les autres services.

## Configuration

```yaml
server:
  port: 8081

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/rofecare_identity
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
  data:
    redis:
      host: localhost
      port: 6379
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

jwt:
  secret: ${JWT_SECRET}
  access-token-expiration: 3600000    # 1 heure en ms
  refresh-token-expiration: 86400000  # 24 heures en ms

security:
  password:
    min-length: 8
    max-failed-attempts: 5
```

## Schema de base de donnees (tables principales)

| Table | Description |
|---|---|
| `users` | Comptes utilisateurs (username, email, password hash, statut) |
| `roles` | Roles du systeme (nom, description) |
| `permissions` | Permissions atomiques (nom, description, ressource) |
| `user_roles` | Table de jointure utilisateur-role |
| `role_permissions` | Table de jointure role-permission |
| `refresh_tokens` | Tokens de rafraichissement actifs |
| `login_attempts` | Historique des tentatives de connexion |

## Tests

Le service utilise :

- **JUnit 5** pour les tests unitaires
- **Mockito** pour le mocking des dependances
- **Testcontainers** pour les tests d'integration (PostgreSQL, Redis)
- Les tests couvrent les couches domain, application et infrastructure
