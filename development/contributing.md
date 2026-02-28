# Guide de contribution

Directives et conventions pour contribuer au projet Rofecare HIS. Ce document decrit le workflow Git, les standards de code et les processus de revue.

---

## Table des matieres

- [Workflow Git Flow](#workflow-git-flow)
- [Conventions de nommage des branches](#conventions-de-nommage-des-branches)
- [Conventional Commits](#conventional-commits)
- [Processus de Pull Request](#processus-de-pull-request)
- [Revue de code](#revue-de-code)
- [Standards de code](#standards-de-code)
- [Exigences de tests](#exigences-de-tests)
- [Exigences de documentation](#exigences-de-documentation)

---

## Workflow Git Flow

Le projet suit le modele **Git Flow** avec les branches suivantes :

```
main (production)
 │
 ├── hotfix/*      ← correctifs urgents en production
 │
develop (integration)
 │
 ├── feature/*     ← nouvelles fonctionnalites
 ├── bugfix/*      ← corrections de bugs
 └── release/*     ← preparation des releases
```

### Branches principales

| Branche   | Description                                | Protection        |
| --------- | ------------------------------------------ | ------------------ |
| `main`    | Code en production, toujours deployable    | Protegee, merge via PR uniquement |
| `develop` | Branche d'integration, prochaine release   | Protegee, merge via PR uniquement |

### Branches de travail

| Type       | Prefix      | Base       | Merge vers  | Usage                          |
| ---------- | ----------- | ---------- | ----------- | ------------------------------ |
| Feature    | `feature/`  | `develop`  | `develop`   | Nouvelles fonctionnalites      |
| Bugfix     | `bugfix/`   | `develop`  | `develop`   | Corrections de bugs            |
| Hotfix     | `hotfix/`   | `main`     | `main` + `develop` | Correctifs urgents production |
| Release    | `release/`  | `develop`  | `main` + `develop` | Preparation de release         |

### Workflow typique

```bash
# 1. Creer une branche feature depuis develop
git checkout develop
git pull origin develop
git checkout -b feature/patient-search-filter

# 2. Travailler et commiter
git add .
git commit -m "feat(patient): add search filter by date of birth"

# 3. Pousser la branche
git push -u origin feature/patient-search-filter

# 4. Creer une Pull Request vers develop
# (via l'interface GitHub)

# 5. Apres la revue et le merge, supprimer la branche
git checkout develop
git pull origin develop
git branch -d feature/patient-search-filter
```

---

## Conventions de nommage des branches

### Format

```
<type>/<description-courte>
```

### Regles

- Utiliser des minuscules et des tirets (`-`) comme separateurs
- La description doit etre courte et descriptive (3-5 mots max)
- Inclure le numero du ticket si applicable

### Exemples

```
feature/patient-search-filter
feature/lab-results-export
bugfix/login-token-expiry
bugfix/prescription-dosage-validation
hotfix/critical-auth-bypass
release/1.2.0
```

---

## Conventional Commits

Tous les messages de commit doivent suivre le format **Conventional Commits**.

### Format

```
<type>(<scope>): <description>

[body optionnel]

[footer optionnel]
```

### Types

| Type       | Description                                              | Exemple                                                |
| ---------- | -------------------------------------------------------- | ------------------------------------------------------ |
| `feat`     | Nouvelle fonctionnalite                                  | `feat(patient): add patient photo upload`              |
| `fix`      | Correction de bug                                        | `fix(clinical): resolve null pointer in diagnosis`     |
| `docs`     | Documentation uniquement                                 | `docs(identity): update API endpoint descriptions`     |
| `style`    | Formatage, points-virgules manquants, etc.               | `style(pharmacy): fix indentation in service layer`    |
| `refactor` | Refactoring sans changement fonctionnel                  | `refactor(finance): extract billing calculator`        |
| `test`     | Ajout ou modification de tests                           | `test(patient): add unit tests for search service`     |
| `chore`    | Maintenance, dependances, configuration                  | `chore(infrastructure): update PostgreSQL to 16.2`     |
| `perf`     | Amelioration des performances                            | `perf(clinical): optimize diagnosis query with index`  |
| `ci`       | Configuration CI/CD                                      | `ci(identity): add Docker build step`                  |

### Scopes

| Scope                | Service / Module                        |
| -------------------- | --------------------------------------- |
| `identity`           | Identity & Access Service               |
| `patient`            | Patient Service                         |
| `clinical`           | Clinical Service                        |
| `medical-technology` | Medical Technology Service              |
| `pharmacy`           | Pharmacy Service                        |
| `finance`            | Finance Service                         |
| `platform`           | Platform Service                        |
| `interoperability`   | Interoperability Service                |
| `common`             | rofecare-common (module partage)        |
| `server`             | rofecare-server (monolithe)             |
| `gateway`            | API Gateway                             |
| `infrastructure`     | Docker, CI/CD, configuration            |

### Exemples complets

```bash
# Nouvelle fonctionnalite
git commit -m "feat(patient): add advanced search with date range filter

Implement patient search filtering by date of birth range.
Includes pagination and sorting support.

Closes #142"

# Correction de bug
git commit -m "fix(clinical): prevent duplicate diagnosis entries

Check for existing diagnosis before inserting.
Added unique constraint on (patient_id, icd_code, date).

Fixes #256"

# Refactoring
git commit -m "refactor(pharmacy): extract inventory validation logic

Move stock validation from controller to dedicated domain service.
No functional change."

# Changement majeur (breaking change)
git commit -m "feat(identity)!: migrate to OAuth2 authorization server

BREAKING CHANGE: JWT token format has changed.
All clients must update their token parsing logic.
See migration guide in docs/migration-oauth2.md"
```

### Breaking Changes

Les changements non retrocompatibles sont signales par :

- Un `!` apres le scope : `feat(identity)!: ...`
- Un footer `BREAKING CHANGE:` dans le corps du commit

---

## Processus de Pull Request

### Creer une Pull Request

1. Poussez votre branche vers le remote
2. Ouvrez une PR via l'interface GitHub
3. Remplissez le template de PR :

```markdown
## Description
Description claire du changement et de sa motivation.

## Type de changement
- [ ] Nouvelle fonctionnalite (feat)
- [ ] Correction de bug (fix)
- [ ] Refactoring (refactor)
- [ ] Documentation (docs)
- [ ] Autre (preciser)

## Checklist
- [ ] Le code suit les standards du projet
- [ ] Les tests unitaires sont ecrits et passent
- [ ] Les tests d'integration passent
- [ ] La documentation API est a jour
- [ ] Les tests ArchUnit passent
- [ ] Pas de warnings de compilation

## Comment tester
Instructions pour tester le changement manuellement.

## Screenshots (si applicable)
```

### Criteres de merge

Une PR peut etre mergee lorsque :

- Tous les checks CI sont au vert
- Au moins 1 revue approuvee
- Aucune conversation non resolue
- La branche est a jour avec la branche cible

### Strategie de merge

- **Feature -> Develop** : Squash and merge (un seul commit propre)
- **Release -> Main** : Merge commit (conserver l'historique)
- **Hotfix -> Main** : Merge commit

---

## Revue de code

### Ce que le reviewer verifie

| Aspect               | Points de verification                                   |
| -------------------- | -------------------------------------------------------- |
| Architecture         | Respect de l'architecture hexagonale                     |
| Domaine              | Pas de dependances Spring dans le package `domain`       |
| Tests                | Couverture suffisante, cas limites testes                |
| Securite             | Pas de donnees sensibles, validation des entrees         |
| Performance          | Requetes N+1, index manquants, pagination                |
| Nommage              | Conventions de nommage respectees                        |
| API                  | Coherence avec les endpoints existants                   |
| Commits              | Format Conventional Commits respecte                     |

### Bonnes pratiques pour les reviewers

- Etre constructif et bienveillant
- Proposer des alternatives, pas seulement des critiques
- Distinguer les blockers des suggestions (prefixer avec `nit:` pour les suggestions mineures)
- Repondre dans les 24h ouvrables

### Bonnes pratiques pour les auteurs

- Garder les PRs petites et focalisees (< 400 lignes idealement)
- Fournir un contexte suffisant dans la description
- Repondre a tous les commentaires
- Ne pas forcer le merge si des questions sont en suspens

---

## Standards de code

### Architecture hexagonale

| Regle                                                              | Verification         |
| ------------------------------------------------------------------ | -------------------- |
| `domain/` ne depend pas de Spring, JPA ou tout autre framework     | ArchUnit             |
| `domain/` ne depend pas de `infrastructure/`                       | ArchUnit             |
| Les use cases implementent les ports IN                            | Revue de code        |
| Les repositories implementent les ports OUT                        | Revue de code        |
| Les DTOs REST sont dans `infrastructure/adapter/in/rest/`          | Convention           |
| Les entites JPA sont dans `infrastructure/adapter/out/persistence/` | Convention          |

### MapStruct pour le mapping

Utilisez **MapStruct** pour tous les mappings entre couches :

```java
@Mapper(componentModel = "spring")
public interface PatientMapper {
    Patient toDomain(PatientEntity entity);
    PatientEntity toEntity(Patient domain);
    PatientResponse toResponse(Patient domain);
}
```

> **Interdit** : le mapping manuel dans les controllers ou services (sauf cas trivial).

### Conventions de nommage

| Element              | Convention           | Exemple                          |
| -------------------- | -------------------- | -------------------------------- |
| Package              | lowercase            | `com.rofecare.patient.domain`    |
| Classe               | PascalCase           | `PatientService`                 |
| Interface            | PascalCase           | `RegisterPatientUseCase`         |
| Methode              | camelCase            | `findByDateOfBirth()`            |
| Constante            | UPPER_SNAKE_CASE     | `MAX_RETRY_COUNT`                |
| Variable             | camelCase            | `patientCount`                   |
| Endpoint REST        | kebab-case           | `/api/v1/patient-records`        |
| Table SQL            | snake_case           | `patient_record`                 |
| Colonne SQL          | snake_case           | `date_of_birth`                  |
| Topic Kafka          | dot.notation         | `patient.registered`             |
| Fichier YAML         | kebab-case           | `application-local.yml`          |

### Structure des endpoints REST

```
/api/v1/{service}/{resource}
/api/v1/{service}/{resource}/{id}
/api/v1/{service}/{resource}/{id}/{sub-resource}
```

### Gestion des erreurs

Utilisez les exceptions de domaine et le handler global :

```java
// Exception de domaine
public class PatientNotFoundException extends RuntimeException {
    public PatientNotFoundException(PatientId id) {
        super("Patient not found: " + id.value());
    }
}

// Handler global (infrastructure)
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(PatientNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(PatientNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("PATIENT_NOT_FOUND", ex.getMessage()));
    }
}
```

---

## Exigences de tests

### Couverture minimale

| Type de test           | Couverture requise | Emplacement                   |
| ---------------------- | ------------------ | ----------------------------- |
| Tests unitaires domaine | 90%+              | `test/.../domain/`            |
| Tests use cases        | 80%+               | `test/.../application/`       |
| Tests d'integration    | Endpoints critiques | `test/.../infrastructure/`   |
| Tests ArchUnit         | Obligatoires        | `test/.../architecture/`     |

### Conventions de test

```java
// Nommage des methodes de test
@Test
void shouldRegisterNewPatient_whenValidData() { }

@Test
void shouldThrowException_whenPatientNotFound() { }

@Test
void shouldReturnPagedResults_whenSearchingByName() { }
```

### Structure des tests

```java
@Test
void shouldRegisterNewPatient_whenValidData() {
    // Given (Arrange)
    RegisterPatientCommand command = RegisterPatientCommand.builder()
        .firstName("Jean")
        .lastName("Dupont")
        .build();

    // When (Act)
    PatientId result = useCase.register(command);

    // Then (Assert)
    assertThat(result).isNotNull();
    verify(patientRepository).save(any(Patient.class));
}
```

### Outils de test

| Outil           | Usage                                        |
| --------------- | -------------------------------------------- |
| JUnit 5         | Framework de test                            |
| Mockito         | Mocking des dependances                      |
| AssertJ         | Assertions fluides                           |
| Testcontainers  | Bases de donnees pour tests d'integration    |
| ArchUnit        | Verification de l'architecture               |
| REST Assured    | Tests d'API REST                             |

---

## Exigences de documentation

### Code

- Les classes publiques des ports IN doivent avoir une Javadoc decrivant le cas d'usage
- Les endpoints REST doivent avoir des annotations OpenAPI (`@Operation`, `@ApiResponse`)
- Les methodes non triviales dans le domaine doivent avoir une Javadoc concise

### API

- Toute modification d'un endpoint REST doit mettre a jour les annotations SpringDoc
- Les DTOs de requete et reponse doivent avoir des annotations de validation documentees

### Commits et PRs

- Chaque commit suit le format Conventional Commits
- Chaque PR a une description claire et complete
- Les breaking changes sont documentes dans le commit et la PR
