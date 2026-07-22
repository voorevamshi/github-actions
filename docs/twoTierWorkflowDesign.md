two-tier workflow design—where your individual microservice repositories (`ci.yml`) call your central shared pipeline (`reusable-maven-docker.yml`).

### The Reusable Pipeline Architecture Map

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CALLER WORKFLOW (ci.yml)                        │
│                Located in: /product-service/.github/workflows/         │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ Triggers on: git push / PR
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                       REUSABLE WORKFLOW PIPELINE                       │
│      Located in: /central-workflows/.github/workflows/reusable-mvn.yml │
└────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ├──► 1. Set up JDK & Maven Credentials
                                   │    (actions/setup-java@v4 with server-id: github)
                                   │
                                   ├──► 2. Execute Single Build & Deploy Step
                                   │    `mvn clean deploy -DsuppressCentralAuth=true`
                                   │    │
                                   │    ├──► [Local Workspace] target/*.jar created
                                   │    │
                                   │    └──► [GitHub Packages] Pushes timestamped JAR
                                   │         (com/demo/product-service/1.0.0-SNAPSHOT)
                                   │
                                   ├──► 3. Log in to Docker Hub
                                   │    (docker/login-action@v3 as vamshivoore)
                                   │
                                   └──► 4. Build & Push Container
                                        (docker/build-push-action@v5)
                                        Reads: local `target/*.jar`
                                        Pushes: vamshivoore/product-service:latest

```

### 1. Caller Workflow (`.github/workflows/ci.yml`)

Located in your microservice repository (e.g., `product-service`). It acts as a lightweight trigger that passes repository-specific inputs to the central workflow.

YAML

```
name: CI Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  call-reusable-pipeline:
    permissions:
      contents: read
      packages: write # Crucial for GitHub Packages deployment!

    uses: VamshiOrganization/central-workflows/.github/workflows/reusable-maven-docker.yml@main
    with:
      java-version: '21'
      image-name: ${{ github.event.repository.name }}
    secrets:
      DOCKERHUB_TOKEN: ${{ secrets.DOCKERHUB_TOKEN }}
      GH_PACKAGES_TOKEN: ${{ secrets.GITHUB_TOKEN }}

```

### 2. Reusable Workflow (`reusable-maven-docker.yml`)

Hosted in a centralized GitHub repository. It contains the standardized build, packaging, and container publishing rules used across all your Java microservices.

YAML

```
name: Reusable Maven & Docker Pipeline

on:
  workflow_call:
    inputs:
      java-version:
        required: false
        type: string
        default: '21'
      image-name:
        required: true
        type: string

    secrets:
      DOCKERHUB_TOKEN:
        required: true
      GH_PACKAGES_TOKEN:
        required: true

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Check out microservice code
        uses: actions/checkout@v4

      - name: Set up JDK ${{ inputs.java-version }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ inputs.java-version }}
          distribution: 'temurin'
          server-id: github # Configures Maven auth for GitHub Packages
          cache: 'maven'

      - name: Build, Test & Deploy to GitHub Packages
        run: mvn clean deploy -DsuppressCentralAuth=true
        env:
          GITHUB_TOKEN: ${{ secrets.GH_PACKAGES_TOKEN }}

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: vamshivoore
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            vamshivoore/${{ inputs.image-name }}:latest

```

### Key Takeaways of This Design

1.  **Zero Duplication:** If you need to change JDK versions, update Maven flags (like adding `-DsuppressCentralAuth=true`), or switch Docker registries, you change it **once** in `reusable-maven-docker.yml`. All microservices inherit the fix instantly.
    
2.  **Context Retention:** The `docker/build-push-action` runs on the same GitHub Actions runner where `mvn clean deploy` generated the file, ensuring `COPY target/*.jar app.jar` inside the Dockerfile finds the artifact without needing cross-job uploads or artifacts download steps.
