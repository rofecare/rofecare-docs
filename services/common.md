# Common Library (rofecare-common)

> **Type** : Bibliotheque partagee (Maven module)
> **Module Maven** : `rofecare-common`
> **Package racine** : `com.rofecare.common`
> **Tests unitaires** : 244
> **Port** : Aucun (non deployable)

## Vue d'ensemble

`rofecare-common` est une bibliotheque partagee importee comme dependance Maven par tous les services de l'ecosysteme Rofecare. Elle ne constitue pas un service deployable mais fournit les composants transversaux essentiels : DTOs partages, utilitaires, securite JWT, configuration commune, gestion des exceptions, integration Kafka et entite de base pour la persistence.

Cette bibliotheque garantit la coherence entre tous les services et elimine la duplication de code pour les preoccupations transversales.

## Dependance Maven

```xml
<dependency>
    <groupId>com.rofecare</groupId>
    <artifactId>rofecare-common</artifactId>
    <version>${rofecare.version}</version>
</dependency>
```

## Structure des packages

```
com.rofecare.common/
├── dto/                  # DTOs partages
├── util/                 # Utilitaires
├── security/             # Securite JWT
├── config/               # Configurations partagees
├── exception/            # Gestion globale des exceptions
├── event/                # Integration Kafka
└── persistence/          # Entite de base
```

---

## Package `dto` - DTOs partages

Les DTOs partages definissent les structures de donnees communes utilisees a travers tous les services, garantissant une representation uniforme des concepts transversaux.

| Classe | Description |
|--------|-------------|
| `Address` | Adresse postale complete (rue, ville, code postal, pays). Utilise comme Value Object dans les entites Patient, Organization, etc. |
| `Email` | Value Object encapsulant une adresse email avec validation du format. |
| `PhoneNumber` | Value Object encapsulant un numero de telephone avec indicatif pays et validation. |
| `Money` | Representation monetaire avec montant (`BigDecimal`) et devise (`Currency`). Utilise dans tout le module Finance. |
| `DateRange` | Plage de dates (debut/fin) utilisee pour les filtres de recherche et les periodes de planification. |
| `PageRequest` | Requete de pagination standardisee : numero de page, taille, champ de tri, direction. |
| `PageResponse<T>` | Reponse paginee generique : contenu, numero de page, taille totale, nombre total de pages. |

### Exemple d'utilisation

```java
// Dans un controller de n'importe quel service
@GetMapping("/patients")
public PageResponse<PatientDto> list(PageRequest pageRequest) {
    return patientService.findAll(pageRequest);
}
```

---

## Package `util` - Utilitaires

| Classe | Description |
|--------|-------------|
| `DateTimeUtil` | Utilitaires de manipulation de dates : conversion de fuseaux horaires, formatage, calcul de durees, extraction de periodes. |
| `ValidationUtil` | Methodes de validation metier : validation d'email, de telephone, de numero national d'identification, de codes CIM-10, etc. |
| `MaskingUtil` | Utilitaire de masquage de donnees sensibles pour les logs et l'affichage : masquage de numeros de telephone, emails, numeros d'assurance. |
| `SlugUtil` | Generation de slugs URL-friendly a partir de chaines de caracteres. Utilise pour les identifiants lisibles (codes de service, etc.). |

### Exemple d'utilisation

```java
// Masquage pour les logs
String masked = MaskingUtil.maskEmail("patient@example.com");
// Resultat : "p*****t@example.com"

String maskedPhone = MaskingUtil.maskPhone("+243812345678");
// Resultat : "+243****5678"
```

---

## Package `security` - Securite JWT

Ce package constitue le coeur du systeme de securite de Rofecare. Il fournit l'authentification et l'autorisation basees sur JWT, utilisees de maniere uniforme par tous les services.

| Classe | Description |
|--------|-------------|
| `JwtTokenProvider` | Gestion du cycle de vie des tokens JWT : generation, validation, extraction des claims (userId, roles, permissions). Supporte les tokens d'acces et de rafraichissement. |
| `JwtAuthenticationFilter` | Filtre Spring Security interceptant chaque requete HTTP pour extraire et valider le token JWT du header `Authorization: Bearer <token>`. |
| `SecurityContextUtil` | Utilitaire statique pour acceder au contexte de securite courant : ID de l'utilisateur connecte, ses roles, ses permissions, son etablissement. |
| `@HasPermission` | Annotation personnalisee pour le controle d'acces declaratif au niveau des methodes. Supporte les expressions combinant permissions et roles. |
| `PermissionAspect` | Aspect AOP evaluant l'annotation `@HasPermission` avant l'execution de la methode. Leve une `ForbiddenException` si les permissions sont insuffisantes. |

### Annotation `@HasPermission`

```java
// Controle d'acces declaratif
@HasPermission("PATIENT_READ")
@GetMapping("/patients/{id}")
public PatientDto getPatient(@PathVariable UUID id) {
    return patientService.findById(id);
}

// Permissions multiples (toutes requises)
@HasPermission({"INVOICE_CREATE", "FINANCE_ACCESS"})
@PostMapping("/invoices")
public InvoiceDto createInvoice(@RequestBody CreateInvoiceRequest request) {
    return invoiceService.create(request);
}
```

### Flux d'authentification

```
Client -> [Authorization: Bearer <JWT>]
    -> JwtAuthenticationFilter
        -> JwtTokenProvider.validateToken()
        -> JwtTokenProvider.extractClaims()
        -> SecurityContextHolder.setAuthentication()
    -> @HasPermission verification via PermissionAspect
    -> Controller method execution
```

---

## Package `config` - Configurations partagees

Les configurations partagees sont auto-configurees via Spring Boot et appliquees automatiquement lorsqu'un service inclut la dependance `rofecare-common`.

| Classe | Description |
|--------|-------------|
| `JacksonConfig` | Configuration de la serialisation JSON : format de date ISO-8601, gestion des nulls, modules JavaTime et Kotlin. |
| `KafkaConfig` | Configuration de base Kafka : serializers/deserializers, error handlers, retry policy. Chaque service complete avec son propre `group-id`. |
| `RedisConfig` | Configuration du client Redis pour le cache distribue et la gestion des sessions. |
| `FeignConfig` | Configuration des clients Feign : intercepteur JWT (propagation du token), error decoder, timeouts, retry. |
| `CorsConfig` | Configuration CORS centralisee : origines autorisees, methodes, headers. Configurable par environnement. |
| `JpaAuditingConfiguration` | Active l'audit JPA automatique : remplissage des champs `createdBy`, `updatedBy`, `createdAt`, `updatedAt` via `AuditorAware`. |

### Propagation JWT via Feign

```java
// FeignConfig injecte automatiquement le token JWT courant dans les appels inter-services
@Bean
public RequestInterceptor jwtRequestInterceptor() {
    return requestTemplate -> {
        String token = SecurityContextUtil.getCurrentToken();
        if (token != null) {
            requestTemplate.header("Authorization", "Bearer " + token);
        }
    };
}
```

---

## Package `exception` - Gestion globale des exceptions

Le package fournit un systeme uniforme de gestion des erreurs pour tous les services, garantissant des reponses d'erreur coherentes au format JSON.

| Classe | Description |
|--------|-------------|
| `GlobalExceptionHandler` | Handler `@RestControllerAdvice` capturant toutes les exceptions et les convertissant en reponses HTTP structurees avec code d'erreur, message et details. |
| `BusinessRuleViolationException` | Exception metier levee lorsqu'une regle de gestion est violee (ex: prescription sans consultation active). HTTP 422. |
| `ForbiddenException` | Exception levee lorsqu'un utilisateur tente une action non autorisee. HTTP 403. |
| `UnauthorizedException` | Exception levee lorsque l'authentification echoue ou que le token est invalide. HTTP 401. |
| `ServiceUnavailableException` | Exception levee lorsqu'un service dependant est injoignable. HTTP 503. |

### Format de reponse d'erreur

```json
{
  "timestamp": "2026-01-15T10:30:00Z",
  "status": 422,
  "error": "Business Rule Violation",
  "code": "PRESCRIPTION_NO_ACTIVE_CONSULTATION",
  "message": "Cannot create a prescription without an active consultation",
  "path": "/api/pharmacy/prescriptions",
  "details": {
    "patientId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

---

## Package `event` - Integration Kafka

Ce package abstrait l'integration Kafka derriere une interface, permettant au mode monolithe de remplacer Kafka par un event bus en memoire.

| Classe | Description |
|--------|-------------|
| `EventPublisher` | Interface definissant le contrat de publication d'evenements. Methode principale : `publish(String topic, Object event)`. |
| `KafkaEventPublisher` | Implementation de `EventPublisher` utilisant Kafka. Active par defaut en mode microservices. |
| `EventMetadata` | Metadonnees attachees a chaque evenement : ID de correlation, timestamp, ID utilisateur source, nom du service emetteur. |

### Exemple d'utilisation

```java
@Service
@RequiredArgsConstructor
public class PatientApplicationService {
    private final EventPublisher eventPublisher;

    public PatientDto register(CreatePatientRequest request) {
        Patient patient = patientDomainService.register(request);

        eventPublisher.publish("patient.registered", new PatientRegisteredEvent(
            patient.getId(),
            patient.getFullName(),
            EventMetadata.current()
        ));

        return patientMapper.toDto(patient);
    }
}
```

---

## Package `persistence` - Entite de base

| Classe | Description |
|--------|-------------|
| `AuditableEntity` | Classe abstraite de base pour toutes les entites JPA de Rofecare. Fournit les champs d'audit et de synchronisation. |

### Champs de `AuditableEntity`

| Champ | Type | Description |
|-------|------|-------------|
| `id` | `UUID` | Identifiant unique genere automatiquement |
| `createdBy` | `String` | ID de l'utilisateur ayant cree l'entite (rempli automatiquement via JPA Auditing) |
| `updatedBy` | `String` | ID du dernier utilisateur ayant modifie l'entite |
| `createdAt` | `Instant` | Date/heure de creation |
| `updatedAt` | `Instant` | Date/heure de derniere modification |
| `optLockVersion` | `Long` | Version pour le verrouillage optimiste (`@Version`), previent les modifications concurrentes |
| `nodeId` | `String` | Identifiant du noeud ayant effectue la modification (utilise pour la synchronisation monolithe-cloud) |
| `syncVersion` | `Long` | Version de synchronisation incrementee a chaque modification (utilise par le mecanisme de sync) |

### Exemple d'utilisation

```java
@Entity
@Table(name = "patients")
public class PatientEntity extends AuditableEntity {

    @Column(nullable = false)
    private String firstName;

    @Column(nullable = false)
    private String lastName;

    @Column(unique = true)
    private String nationalId;

    // ... autres champs specifiques
}
```

---

## Resume des tests

Avec 244 tests unitaires, `rofecare-common` est le troisieme module le plus teste du projet. Les tests couvrent :

- **Security** : Validation JWT, extraction des claims, filtre d'authentification, evaluation des permissions
- **DTOs** : Validation des Value Objects (Email, PhoneNumber, Money), serialisation/deserialisation JSON
- **Utils** : Formatage de dates, validation, masquage de donnees, generation de slugs
- **Exception handling** : Conversion des exceptions en reponses HTTP, format des messages d'erreur
- **Event publishing** : Publication d'evenements, enrichissement des metadonnees
- **Persistence** : Audit automatique (createdBy, updatedAt), verrouillage optimiste
