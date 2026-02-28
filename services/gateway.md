# API Gateway

> **Module** : `rofecare-gateway`
> **Port** : `8080`
> **Framework** : Spring Cloud Gateway (reactive, non-blocking)

## Vue d'ensemble

L'API Gateway constitue le point d'entree unique de la plateforme Rofecare. Construit sur Spring Cloud Gateway en mode reactif (Project Reactor / Netty), il assure le routage dynamique des requetes vers les microservices, l'authentification JWT, la limitation de debit, la gestion CORS et la resilience via des circuit breakers.

Toutes les requetes clientes passent obligatoirement par le gateway, qui decouvre automatiquement les services enregistres aupres d'Eureka.

## Table de routage

| Route | Service cible | Port |
|---|---|---|
| `/api/identity/**` | Identity Service | 8081 |
| `/api/patients/**` | Patient Service | 8082 |
| `/api/clinical/**` | Clinical Service | 8083 |
| `/api/medical-technology/**` | Medical Technology Service | 8084 |
| `/api/pharmacy/**` | Pharmacy Service | 8085 |
| `/api/finance/**` | Finance Service | 8086 |
| `/api/platform/**` | Platform Service | 8087 |
| `/api/interoperability/**` | Interoperability Service | 8088 |

Les routes sont decouvertes automatiquement via Eureka. Chaque service s'enregistre avec son identifiant et le gateway resout les instances disponibles pour le load balancing.

## Filtres

### Filtre d'authentification JWT

Le gateway intercepte chaque requete entrante pour valider le token JWT presente dans le header `Authorization` (format `Bearer <token>`). Les requetes sans token valide sont rejetees avec un code `401 Unauthorized`, a l'exception des endpoints publics (login, register, health checks).

### Filtres de routage

Chaque route applique des filtres Spring Cloud Gateway standards :

- **StripPrefix** : supprime le prefixe de route avant de transmettre la requete au service cible.
- **AddRequestHeader** : injecte des headers de correlation (`X-Correlation-Id`, `X-Trace-Id`) pour le tracing distribue.
- **CircuitBreaker** : encapsule chaque route dans un circuit breaker Resilience4j.

## Rate Limiting

La limitation de debit est assuree par le filtre `RequestRateLimiter` adosse a Redis.

| Parametre | Valeur |
|---|---|
| Debit autorise | 100 requetes/seconde |
| Burst autorise | 200 requetes |
| Backend | Redis |
| Cle de resolution | Adresse IP du client |

```yaml
spring:
  cloud:
    gateway:
      filter:
        request-rate-limiter:
          redis-rate-limiter:
            replenish-rate: 100
            burst-capacity: 200
```

Lorsque la limite est atteinte, le gateway renvoie un code `429 Too Many Requests`.

## Configuration CORS

Le gateway autorise les origines front-end en developpement :

| Parametre | Valeur |
|---|---|
| Origines autorisees | `http://localhost:3000`, `http://localhost:3001` |
| Methodes autorisees | `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS` |
| Headers autorises | `*` |
| Headers exposes | `Authorization`, `X-Correlation-Id`, `X-Trace-Id`, `X-Total-Count`, `X-Page-Number`, `X-Page-Size` |
| Credentials | `true` |

Les headers exposes permettent au front-end d'acceder aux informations de pagination et de tracing dans les reponses.

## Circuit Breaker

Chaque route du gateway est protegee par un circuit breaker Resilience4j. En cas de defaillance d'un service en aval, le circuit s'ouvre pour eviter la propagation des erreurs.

| Parametre | Valeur |
|---|---|
| Taux d'echec pour ouverture | 50% |
| Duree d'attente en etat ouvert | 10 secondes |
| Taille de la fenetre glissante | 10 appels |
| Nombre minimum d'appels | 5 |
| Timeout par appel | Configure par service |

Lorsqu'un circuit s'ouvre, les requetes vers le service concerne recoivent une reponse `503 Service Unavailable` jusqu'a la fermeture du circuit.

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
        sliding-window-size: 10
        minimum-number-of-calls: 5
```

## Securite

Le gateway centralise la securite de la plateforme :

- **Authentification JWT** : validation du token Bearer sur chaque requete (sauf endpoints publics).
- **Rate limiting** : protection contre les abus et les attaques par deni de service.
- **CORS** : restriction des origines autorisees pour prevenir les requetes cross-origin non souhaitees.
- **Circuit breakers** : isolation des defaillances pour proteger la stabilite globale de la plateforme.

Les tokens JWT sont emis par l'Identity Service (port 8081) et valides par le filtre du gateway avant toute transmission vers les services en aval.

## Endpoints Actuator

| Endpoint | Description |
|---|---|
| `/actuator/health` | Etat de sante du gateway |
| `/actuator/info` | Informations sur le service |
| `/actuator/metrics` | Metriques Micrometer |
| `/actuator/gateway/routes` | Liste des routes configurees |
