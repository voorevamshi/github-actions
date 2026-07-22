### 1. Maven Distribution Management (`pom.xml`)

This block tells Maven **where** to publish compiled artifacts (`.jar` / `.war` files).

XML

```
<distributionManagement>
  <repository>
    <id>github</id>
    <name>GitHub Packages</name>
    <url>https://maven.pkg.github.com/VamshiOrganization/${project.artifactId}</url>
  </repository>
</distributionManagement>
```
### Maven `<distributionManagement>` Configuration Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`<distributionManagement>`** | Parent element that declares this project produces deployable artifacts (such as JARs) intended for a remote package registry. |
| **`<repository>`** | Defines the **release repository** configuration. (Use `<snapshotRepository>` separately for development or snapshot builds.) |
| **`<id>github</id>`** | **Critical mapping key.** Must match the `<server><id>github</id></server>` entry in Maven's `settings.xml`, allowing Maven to locate the correct authentication credentials. |
| **`<name>GitHub Packages</name>`** | A human-readable name displayed in Maven logs and build reports for easier identification. |
| **`<url>`** | The HTTPS endpoint where the generated `.jar` artifact is uploaded. Using `${project.artifactId}` dynamically inserts the repository name (for example, `inventory-service`), making the configuration reusable across multiple microservices. |


### 2. Caller Workflow (`.github/workflows/ci.yml`)

Placed in each microservice repository (like `inventory-service`).
```
name: CI/CD Pipeline

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write

jobs:
  call-shared-workflow:
    uses: VamshiOrganization/github-actions/.github/workflows/reusable-maven-docker.yml@main
    secrets: inherit
```
### GitHub Actions Workflow Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`name:`** | Display name shown on the **GitHub Actions** tab in your repository, making it easy to identify the workflow. |
| **`on.push.branches:`** | Trigger condition. Automatically runs the workflow whenever code is pushed or merged into the specified branch (for example, `main`). |
| **`permissions.contents: read`** | Grants **read-only** access to the source code repository. Follows the **Principle of Least Privilege** by providing only the minimum required access. |
| **`permissions.packages: write`** | Grants the `GITHUB_TOKEN` permission to publish packages to **GitHub Packages** or other supported package registries. |
| **`jobs.call-shared-workflow`** | The job identifier within the workflow. This job executes a reusable workflow instead of defining all steps directly. |
| **`uses:`** | Imports and invokes a reusable workflow from another repository using the format `owner/repo/path@ref`. Promotes code reuse and centralized CI/CD management. |
| **`secrets: inherit`** | Automatically passes all available repository-level and organization-level secrets (such as `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`) to the reusable workflow, eliminating the need to redefine them. |

### 3. Reusable Central Workflow (`reusable-maven-docker.yml`)

Placed in your central `github-actions` repository.

### A. Workflow Invocation & Inputs
```
on:
  workflow_call:
    inputs:
      java-version:
        type: string
        default: '21'
```
### GitHub Actions Reusable Workflow Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`on.workflow_call`** | Marks the workflow as a **reusable workflow**, allowing it to be invoked by other workflows across repositories or within the same organization. |
| **`inputs.java-version`** | Declares a configurable input parameter that calling workflows can provide, enabling different Java versions without modifying the reusable workflow. |
| **`type: string`** | Specifies the expected data type for the input. Supported types include `string`, `boolean`, and `number`, helping validate the caller's input. |
| **`default: '21'`** | Defines the default Java version to use when the calling workflow does not explicitly provide a value for `java-version`. |

###  B. Background Service Containers
```
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    ports:
      - 3306:3306
    options: >-
      --health-cmd="mysqladmin ping"
      --health-interval=10s
      --health-timeout=5s
      --health-retries=3
```
### GitHub Actions Service Container Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`services:`** | Instructs GitHub Actions to start one or more **Docker service containers** that run alongside the workflow and are connected to the runner's network during job execution. |
| **`mysql.image:`** | Specifies the official Docker image used to create the temporary MySQL database (for example, `mysql:8.0`). |
| **`env.MYSQL_ROOT_PASSWORD`** | Sets the password for the MySQL **root** user inside the Docker container during initialization. |
| **`env.MYSQL_DATABASE`** | Automatically creates the specified database (for example, `testdb`) when the MySQL container starts. |
| **`ports: ["3306:3306"]`** | Maps port **3306** inside the MySQL container to port **3306** on the GitHub Actions runner, allowing applications to connect using `localhost:3306`. |
| **`options: --health-cmd`** | Defines a Docker **health check** command that repeatedly verifies MySQL is fully initialized and ready before the workflow executes database-dependent steps. |
| **`--health-interval=10s`** | Specifies how often Docker runs the health check (every **10 seconds**). |
| **`--health-timeout=5s`** | Sets the maximum time Docker waits for a health check to complete before marking that attempt as failed. |
| **`--health-retries=3`** | Specifies the number of consecutive failed health checks required before Docker marks the MySQL service as **unhealthy**. |

### C. Build & Deployment Steps

#### `actions/setup-java@v4`

```
- name: Set up JDK ${{ inputs.java-version }}
  uses: actions/setup-java@v4
  with:
    java-version: ${{ inputs.java-version }}
    distribution: 'temurin'
    server-id: github
    cache: 'maven'
```
### GitHub Actions Java Setup Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`distribution: 'temurin'`** | Specifies the JDK vendor distribution to install. **Temurin** (formerly AdoptOpenJDK) is the Eclipse Foundation's OpenJDK distribution and is widely used for production workloads. |
| **`server-id: github`** | A critical Maven configuration property. Automatically creates a `~/.m2/settings.xml` file on the GitHub Actions runner with a matching `<server><id>github</id></server>` entry, allowing Maven to authenticate securely with **GitHub Packages** using the workflow token. |
| **`cache: 'maven'`** | Enables Maven dependency caching by preserving the `~/.m2/repository` directory between workflow runs. This significantly reduces build time by avoiding repeated downloads of dependencies. |

#### Maven Execution

YAML

```
- name: Build and Publish to GitHub Packages
  run: mvn clean deploy
  env:
    GITHUB_TOKEN: ${{ github.token }}
```
## Maven Build & Deployment Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`mvn clean`** | Deletes the `target/` directory to remove previous build artifacts, ensuring a clean and uncorrupted compilation before starting a new build. |
| **`mvn deploy`** | Executes the complete Maven deployment lifecycle: compiles the source code, runs unit/integration tests, packages the application into a `.jar`, and uploads the artifact to the repository specified in the `<distributionManagement>` section of `pom.xml`. |
| **`env.GITHUB_TOKEN`** | Injects GitHub's automatically generated, short-lived authentication token into the workflow environment, enabling Maven to securely authenticate and publish artifacts to **GitHub Packages** (`maven.pkg.github.com`). |
#### Docker Build & Push

YAML

```
- name: Build and push Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: |
      ${{ secrets.DOCKERHUB_USERNAME }}/${{ github.event.repository.name }}:latest
      ${{ secrets.DOCKERHUB_USERNAME }}/${{ github.event.repository.name }}:${{ git
```
## Docker Build & Push Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`context: .`** | Specifies the Docker **build context** as the repository root (`.`). Docker uses this directory to locate the `Dockerfile` and all files required during the image build. |
| **`push: true`** | Automatically pushes the successfully built Docker image to **Docker Hub** (or the configured container registry) after the build completes. |
| **`tags:`** | Applies one or more tags to the Docker image. Commonly includes:<br>1. **`latest`** – Represents the most recent production-ready image.<br>2. **`${{ github.sha }}`** – Uses the unique Git commit SHA to create an immutable image version for traceability and reliable rollbacks. |
| **`${{ github.event.repository.name }}`** | Dynamically resolves to the current GitHub repository name (for example, `inventory-service`), allowing the workflow to automatically generate the correct image name for different microservices without hardcoding it. |

### 4. Test Environment Config (`src/test/resources/application.properties`)

Applied specifically when running JUnit integration tests.
```
spring.datasource.url=jdbc:mysql://localhost:3306/testdb?useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
```
## Spring Boot Test Database Configuration Properties Explained

| **Property** | **What It Means & Why It Matters** |
|---|---|
| **`spring.datasource.url`** | The JDBC connection URL that connects the Spring Boot application to the temporary MySQL container running on `localhost:3306` during GitHub Actions execution. |
| **`useSSL=false`** | Disables SSL encryption for local or CI test environments, reducing connection overhead and improving application startup time. |
| **`allowPublicKeyRetrieval=true`** | Allows the MySQL 8.x JDBC driver to retrieve the server's public key automatically during authentication, eliminating the need for manual RSA key configuration in test environments. |
| **`spring.datasource.username` / `spring.datasource.password`** | Database credentials used by Spring Boot to authenticate with MySQL. These values must match the `MYSQL_ROOT_PASSWORD` (or configured user credentials) defined in the GitHub Actions MySQL service container. |
| **`spring.jpa.hibernate.ddl-auto=create-drop`** | Instructs Hibernate to automatically create database tables for all `@Entity` classes when tests start and drop them after the tests finish, ensuring a clean database for every test run. |
| **`spring.jpa.show-sql=true`** | Logs all SQL statements generated by Hibernate to the GitHub Actions console, making it easier to debug database operations and verify generated queries. |
