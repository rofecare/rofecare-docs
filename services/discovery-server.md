# Discovery Server

> **Module** : `rofecare-discovery-server`
> **Port** : `8761`
> **Framework** : Netflix Eureka Server

## Vue d'ensemble

Le Discovery Server est le registre de services de la plateforme Rofecare. Base sur Netflix Eureka, il permet a chaque microservice de s'enregistrer automatiquement au demarrage et de decouvrir les autres services sans configuration d'adresses en dur.

L'API Gateway et tous les services metier s'appuient sur Eureka pour la resolution de noms et le load balancing cote client.

## Services enregistres

Le Discovery Server gere l'enregistrement des services suivants :

| Service | Identifiant Eureka | Port |
|---|---|---|
| API Gateway | `rofecare-gateway` | 8080 |
| Identity Service | `rofecare-service-identity` | 8081 |
| Patient Service | `rofecare-service-patient` | 8082 |
| Clinical Service | `rofecare-service-clinical` | 8083 |
| Medical Technology Service | `rofecare-service-medical-technology` | 8084 |
| Pharmacy Service | `rofecare-service-pharmacy` | 8085 |
| Finance Service | `rofecare-service-finance` | 8086 |
| Platform Service | `rofecare-service-platform` | 8087 |
| Interoperability Service | `rofecare-service-interoperability` | 8088 |

## Enregistrement des services

Chaque microservice s'enregistre aupres d'Eureka au demarrage en incluant la dependance `spring-cloud-starter-netflix-eureka-client` et la configuration suivante :

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

### Processus d'enregistrement

1. Le service demarre et se connecte au Discovery Server.
2. Il envoie un heartbeat initial contenant son identifiant, adresse IP et port.
3. Le Discovery Server l'ajoute a son registre.
4. Le service envoie des heartbeats periodiques toutes les 10 secondes (`lease-renewal-interval-in-seconds: 10`).
5. Si le Discovery Server ne recoit plus de heartbeat pendant 90 secondes, il retire le service du registre.

### Decouverte

Les services clients interrogent le registre Eureka pour obtenir la liste des instances disponibles d'un service cible. Le rafraichissement du registre local se fait a intervalles reguliers (`fetch-registry: true`).

## Configuration

```yaml
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
    prefer-ip-address: true
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    enable-self-preservation: true
```

Le serveur Eureka lui-meme ne s'enregistre pas aupres d'un autre registre (`register-with-eureka: false`) et ne tente pas de recuperer un registre externe (`fetch-registry: false`).

## Dashboard

Le tableau de bord Eureka est accessible a l'adresse :

```
http://localhost:8761
```

Il affiche :
- La liste des services enregistres avec leur statut (UP, DOWN, OUT_OF_SERVICE).
- Le nombre d'instances par service.
- Les informations de self-preservation.
- Les renouvellements de bail recents.
- Les statistiques generales du registre.

## Health Checks

Eureka surveille l'etat de sante de chaque service via les heartbeats periodiques. Lorsqu'un service ne renouvelle plus son bail :

1. Apres 90 secondes sans heartbeat, le service est marque comme expire.
2. Eureka retire l'instance du registre (sauf en mode self-preservation).
3. Les clients qui interrogent le registre ne recoivent plus cette instance.

Les services exposent egalement leurs propres endpoints de sante via Spring Boot Actuator (`/actuator/health`), ce qui permet une verification plus fine de leur disponibilite.

## Mode Self-Preservation

Le mode self-preservation est active par defaut. Lorsque Eureka detecte que le nombre de renouvellements de bail tombe en dessous d'un seuil critique (generalement 85%), il cesse de retirer les services du registre, en partant du principe que c'est un probleme reseau plutot qu'une defaillance massive des services.

Ce mecanisme evite la suppression en cascade de services lors de perturbations reseau temporaires.

## Mode Monolithique

En mode monolithique (deploiement unique sans microservices), le Discovery Server est desactive. Les services communiquent directement entre eux sans passer par le registre Eureka. Ce mode est utile pour :

- Le developpement local simplifie.
- Les environnements avec ressources limitees.
- Les deploiements sur un seul noeud.

La desactivation se fait via la propriete de configuration :

```yaml
eureka:
  client:
    enabled: false
```
