# CI/CD Pipeline

Documentation des pipelines d'integration continue et de deploiement continu de Rofecare HIS, basees sur GitHub Actions.

---

## Table des matieres

- [Vue d'ensemble des workflows](#vue-densemble-des-workflows)
- [ci.yml - Build et tests par service](#ciyml---build-et-tests-par-service)
- [release.yml - Release du serveur monolithe](#releaseyml---release-du-serveur-monolithe)
- [release-desktop.yml - Release de l'application desktop](#release-desktopyml---release-de-lapplication-desktop)
- [Strategie de tagging Docker](#strategie-de-tagging-docker)
- [GitHub Packages / GHCR](#github-packages--ghcr)
- [Regles de protection des branches](#regles-de-protection-des-branches)
- [Configuration Dependabot](#configuration-dependabot)

---

## Vue d'ensemble des workflows

| Workflow             | Fichier               | Declencheur                     | Description                               |
| -------------------- | --------------------- | ------------------------------- | ----------------------------------------- |
| CI (par service)     | `ci.yml`              | Push sur `main`/`develop`, PRs  | Build, tests, Docker push                 |
| Release serveur      | `release.yml`         | Tag `v*.*.*`                    | Release GitHub avec fat JAR               |
| Release desktop      | `release-desktop.yml` | Tag `v*.*.*`                    | Builds multi-plateforme Tauri             |

---

## ci.yml - Build et tests par service

Chaque service possede un workflow CI identique. Voici la structure :

```yaml
name: CI - Patient Service

on:
  push:
    branches: [main, develop]
    paths:
      - "rofecare-patient-service/**"
      - "rofecare-common/**"
  pull_request:
    branches: [main, develop]
    paths:
      - "rofecare-patient-service/**"
      - "rofecare-common/**"

env:
  JAVA_VERSION: "21"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/patient-service

jobs:
  # ============================================================
  # Job 1 : Build & Test
  # ============================================================
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: patient_test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build rofecare-common
        run: |
          cd rofecare-common
          mvn clean install -DskipTests -B

      - name: Build and test service
        run: |
          cd rofecare-patient-service
          mvn clean verify -B
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/patient_test_db
          SPRING_DATASOURCE_USERNAME: test_user
          SPRING_DATASOURCE_PASSWORD: test_password
          TESTCONTAINERS_RYUK_DISABLED: true

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: rofecare-patient-service/target/surefire-reports/

      - name: Upload coverage report
        if: success()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: rofecare-patient-service/target/site/jacoco/

  # ============================================================
  # Job 2 : Build & Push Docker Image
  # ============================================================
  build-and-push-docker:
    name: Build & Push Docker
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build rofecare-common
        run: |
          cd rofecare-common
          mvn clean install -DskipTests -B

      - name: Build service JAR
        run: |
          cd rofecare-patient-service
          mvn clean package -DskipTests -B

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./rofecare-patient-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  # ============================================================
  # Publish artifact to GitHub Packages (Maven)
  # ============================================================
  publish-artifact:
    name: Publish Maven Artifact
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Build rofecare-common
        run: |
          cd rofecare-common
          mvn clean install -DskipTests -B

      - name: Publish to GitHub Packages
        run: |
          cd rofecare-patient-service
          mvn deploy -DskipTests -B
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Services concernes

Ce workflow est duplique pour chaque service avec les adaptations suivantes :

| Service                     | paths filter                              | IMAGE_NAME                    |
| --------------------------- | ----------------------------------------- | ----------------------------- |
| Identity Service            | `rofecare-identity-service/**`            | `.../identity-service`        |
| Patient Service             | `rofecare-patient-service/**`             | `.../patient-service`         |
| Clinical Service            | `rofecare-clinical-service/**`            | `.../clinical-service`        |
| Medical Technology Service  | `rofecare-medical-technology-service/**`  | `.../medical-technology-service` |
| Pharmacy Service            | `rofecare-pharmacy-service/**`            | `.../pharmacy-service`        |
| Finance Service             | `rofecare-finance-service/**`             | `.../finance-service`         |
| Platform Service            | `rofecare-platform-service/**`            | `.../platform-service`        |
| Interoperability Service    | `rofecare-interoperability-service/**`    | `.../interoperability-service`|
| Config Server               | `rofecare-config-server/**`               | `.../config-server`           |
| Discovery Server            | `rofecare-discovery-server/**`            | `.../discovery-server`        |
| Gateway                     | `rofecare-gateway/**`                     | `.../gateway`                 |

> **Note** : le workflow `rofecare-common` ne publie pas d'image Docker mais publie un artefact Maven.

---

## release.yml - Release du serveur monolithe

Ce workflow cree une release GitHub contenant le fat JAR du serveur monolithe :

```yaml
name: Release - Server

on:
  push:
    tags:
      - "v*.*.*"

env:
  JAVA_VERSION: "21"

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest

    permissions:
      contents: write
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: temurin
          cache: maven

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Build rofecare-common
        run: |
          cd rofecare-common
          mvn clean install -DskipTests -B

      - name: Build monolith server
        run: |
          cd rofecare-server
          mvn clean package -DskipTests -B -Pmonolith \
            -Dproject.version=${{ steps.version.outputs.VERSION }}

      - name: Run tests
        run: |
          cd rofecare-server
          mvn verify -B

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          name: Rofecare HIS v${{ steps.version.outputs.VERSION }}
          body: |
            ## Rofecare HIS v${{ steps.version.outputs.VERSION }}

            ### Installation
            ```bash
            java -jar rofecare-server.jar --spring.profiles.active=monolith
            ```

            Consultez le [guide d'installation locale](docs/deployment/local-install.md) pour plus de details.
          files: |
            rofecare-server/target/rofecare-server-*.jar
          draft: false
          prerelease: ${{ contains(steps.version.outputs.VERSION, '-') }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./rofecare-server
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/rofecare-server:${{ steps.version.outputs.VERSION }}
            ghcr.io/${{ github.repository }}/rofecare-server:latest
```

### Creer une release

```bash
# Creer et pousser un tag
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0

# Pour une pre-release
git tag -a v1.3.0-beta.1 -m "Beta release 1.3.0-beta.1"
git push origin v1.3.0-beta.1
```

---

## release-desktop.yml - Release de l'application desktop

Ce workflow compile l'application desktop Tauri pour toutes les plateformes :

```yaml
name: Release - Desktop (Tauri)

on:
  push:
    tags:
      - "v*.*.*"

jobs:
  build-tauri:
    name: Build Desktop App
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macos-latest
            target: aarch64-apple-darwin
            artifact: "*.dmg"
          - platform: macos-13
            target: x86_64-apple-darwin
            artifact: "*.dmg"
          - platform: windows-latest
            target: x86_64-pc-windows-msvc
            artifact: "*.msi"
          - platform: ubuntu-22.04
            target: x86_64-unknown-linux-gnu
            artifact: "*.AppImage,*.deb"

    runs-on: ${{ matrix.platform }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: rofecare-frontend/package-lock.json

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install Linux dependencies
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            libwebkit2gtk-4.1-dev \
            libappindicator3-dev \
            librsvg2-dev \
            patchelf

      - name: Install frontend dependencies
        run: |
          cd rofecare-frontend
          npm ci

      - name: Build frontend
        run: |
          cd rofecare-frontend
          npm run build

      - name: Build Tauri app
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
        with:
          projectPath: rofecare-frontend
          tagName: v__VERSION__
          releaseName: "Rofecare Desktop v__VERSION__"
          releaseBody: |
            Application desktop Rofecare HIS.
            Voir les notes de release du serveur pour les details des changements.
          releaseDraft: false
          prerelease: false
```

### Artefacts produits

| Plateforme | Fichier                           | Architecture |
| ---------- | --------------------------------- | ------------ |
| macOS      | `Rofecare_x.y.z_aarch64.dmg`     | Apple Silicon |
| macOS      | `Rofecare_x.y.z_x64.dmg`         | Intel        |
| Windows    | `Rofecare_x.y.z_x64_en-US.msi`   | x86_64       |
| Linux      | `Rofecare_x.y.z_amd64.AppImage`  | x86_64       |
| Linux      | `rofecare_x.y.z_amd64.deb`       | x86_64       |

---

## Strategie de tagging Docker

Les images Docker sont taguees automatiquement lors du push vers GHCR :

### Tags generes

| Tag                 | Declencheur                  | Exemple                                |
| ------------------- | ---------------------------- | -------------------------------------- |
| `<sha>`             | Tout push sur `main`         | `a1b2c3d`                              |
| `<branch>`          | Tout push sur `main`         | `main`                                 |
| `latest`            | Push sur `main` (default)    | `latest`                               |
| `<version>`         | Tag `v*.*.*`                 | `1.2.0`                                |

### Utilisation

```bash
# Derniere version stable
docker pull ghcr.io/your-org/rofecare-server/patient-service:latest

# Version specifique
docker pull ghcr.io/your-org/rofecare-server/patient-service:1.2.0

# Commit specifique
docker pull ghcr.io/your-org/rofecare-server/patient-service:a1b2c3d
```

### Configuration dans `docker-compose.prod.yml`

```yaml
patient-service:
  image: ghcr.io/your-org/rofecare-server/patient-service:${VERSION:-latest}
```

---

## GitHub Packages / GHCR

### Configuration du repository Maven

Pour publier et consommer les artefacts Maven via GitHub Packages, configurez le `pom.xml` :

```xml
<distributionManagement>
    <repository>
        <id>github</id>
        <name>GitHub Packages</name>
        <url>https://maven.pkg.github.com/your-org/rofecare-server</url>
    </repository>
</distributionManagement>
```

### Authentification locale

Ajoutez dans `~/.m2/settings.xml` :

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>${env.GITHUB_ACTOR}</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

### Images Docker dans GHCR

Les images sont publiees dans GitHub Container Registry (GHCR) :

```
ghcr.io/your-org/rofecare-server/identity-service
ghcr.io/your-org/rofecare-server/patient-service
ghcr.io/your-org/rofecare-server/clinical-service
ghcr.io/your-org/rofecare-server/medical-technology-service
ghcr.io/your-org/rofecare-server/pharmacy-service
ghcr.io/your-org/rofecare-server/finance-service
ghcr.io/your-org/rofecare-server/platform-service
ghcr.io/your-org/rofecare-server/interoperability-service
ghcr.io/your-org/rofecare-server/config-server
ghcr.io/your-org/rofecare-server/discovery-server
ghcr.io/your-org/rofecare-server/gateway
ghcr.io/your-org/rofecare-server/rofecare-server
```

---

## Regles de protection des branches

### Branche `main`

| Regle                                            | Valeur   |
| ------------------------------------------------ | -------- |
| Require pull request before merging              | Oui      |
| Required number of approvals                     | 1        |
| Dismiss stale pull request approvals             | Oui      |
| Require status checks to pass before merging     | Oui      |
| Required status checks                           | `Build & Test` |
| Require branches to be up to date before merging | Oui      |
| Require conversation resolution before merging   | Oui      |
| Do not allow bypassing the above settings        | Oui      |
| Allow force pushes                               | Non      |
| Allow deletions                                  | Non      |

### Branche `develop`

| Regle                                            | Valeur   |
| ------------------------------------------------ | -------- |
| Require pull request before merging              | Oui      |
| Required number of approvals                     | 1        |
| Require status checks to pass before merging     | Oui      |
| Required status checks                           | `Build & Test` |
| Allow force pushes                               | Non      |
| Allow deletions                                  | Non      |

---

## Configuration Dependabot

Dependabot est configure pour maintenir les dependances a jour automatiquement.

### Fichier `.github/dependabot.yml`

```yaml
version: 2
updates:
  # Dependances Maven
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    reviewers:
      - "your-org/backend-team"
    labels:
      - "dependencies"
      - "java"
    groups:
      spring-boot:
        patterns:
          - "org.springframework*"
      testing:
        patterns:
          - "org.junit*"
          - "org.mockito*"
          - "org.testcontainers*"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "ci"

  # Docker
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"
```

### Gestion des PRs Dependabot

- Les PRs Dependabot sont automatiquement labellisees
- La CI s'execute automatiquement sur chaque PR
- Les mises a jour mineures et patch peuvent etre mergees apres revue rapide
- Les mises a jour majeures necessitent une revue approfondie et des tests manuels
