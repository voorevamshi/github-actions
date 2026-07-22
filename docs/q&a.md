

### 1. Workflow Triggers & Branching Strategy

#### **Q: Does my CI/CD pipeline run on every branch, or only on `main`?**

 **Answer:** It depends on your `on:` trigger block. If your `ci.yml` defines `branches: [ "main" ]`, GitHub Action runners will **only** execute on pushes to `main`. To test feature branches or pull requests before merging (while preserving deployments exclusively for `main`), configure wildcard triggers and conditional push steps:
 
 YAML
 
 ```
 on:
   push:
     branches: [ "main", "dev", "feature/**" ]
   pull_request:
     branches: [ "main" ]
 
 ```
 
 Inside the step, use standard GitHub expressions to restrict publishing:
 
 YAML
 
 ```
 push: ${{ github.ref == 'refs/heads/main' }}
 
 ```

### 2. Reusable Workflow Execution Model & Scope

#### **Q: When a caller repo (`inventory-service`) invokes a reusable workflow in another repo (`.github`), how does it know the service context?**

 **Answer:** Reusable workflows act like static helper functions. While the YAML definition is fetched from the central repository (`voorevamshi/.github`), **execution occurs entirely inside the caller repository (`inventory-service`)**.
 
 -   `actions/checkout@v4` pulls the code of the caller repo.
     
 -   `${{ github.event.repository.name }}` evaluates dynamically to `"inventory-service"`.
     
 -   `${{ secrets.DOCKERHUB_USERNAME }}` evaluates using caller/organization secrets.
     

### 3. Maven Dependency & Artifact Publishing Setup

#### **Q: Do I need to keep `<distributionManagement` in my service's `pom.xml` if I use a reusable workflow?**

 **Answer:** **Yes.** Maven requires division of responsibilities:
 
 1.  **`pom.xml` (`<distributionManagement`)**: Tells Maven _where_ to upload the JAR (URL & Server ID).
     
 2.  **GitHub Workflow (`setup-java@v4`)**: Injects credentials into Maven's `settings.xml` so Maven knows _how_ to authenticate.
     
 
 **Pro Tip:** Avoid hardcoding service names in every `pom.xml` by using the built-in Maven property:
 
 XML
 
 ```
 <distributionManagement
   <repository
     <idgithub</id
     <nameGitHub Packages</name
     <urlhttps://maven.pkg.github.com/voorevamshi/${project.artifactId}</url
   </repository
 </distributionManagement
 
 ```

### 4. GitHub Actions Security & Permission Delegation

#### **Q: Why do I get `Error calling workflow: nested job requesting 'packages: write', but is only allowed 'packages: read'`?**

 **Answer:** GitHub Actions enforces strict permission boundaries. A called reusable workflow **cannot elevate permissions beyond what the caller workflow grants it**. Even if `reusable-maven-docker.yml` requests write access, the caller `ci.yml` must explicitly pass `packages: write` at the top level:
 
 YAML
 
 ```
 permissions:
   contents: read
   packages: write # Grants caller and downstream reusable workflow authorization
 
 ```

### 5. Maven Settings & File Path Resolution

#### **Q: Why does Maven fail with `The specified user settings file does not exist: $GITHUB_WORKSPACE/settings.xml`?**

 **Answer:** `actions/setup-java@v4` expects `settings-path` to be a directory, or fails when paths are misaligned with `-s $GITHUB_WORKSPACE/settings.xml`.
 
 **Best Fix:** Omit the custom `-s` flag entirely. `setup-java@v4` automatically injects authenticated GitHub Package credentials into Maven's default `~/.m2/settings.xml` when configured as:
 
 YAML
 
 ```
 - name: Set up JDK 21
   uses: actions/setup-java@v4
   with:
     java-version: '21'
     distribution: 'temurin'
     server-id: github # Automatically configures ~/.m2/settings.xml
 
 - name: Build and Publish
   run: mvn clean deploy # Clean, clean execution
 
 ```

### 6. Containerized Database & Service Integration Testing

#### **Q: Why do Spring Boot tests crash during `mvn clean deploy` with `JDBCConnectionException` / `HikariPool Connection request timed out`?**

 **Answer:** During `mvn clean deploy`, Spring Boot integration tests (`@SpringBootTest`) boot up and look for external services (MySQL on `localhost:3306` or Redis on `localhost:6379`). Because GitHub runners are fresh virtual machines with no DB running, connection attempts time out.

#### **Q: How do I supply MySQL or Redis for pipeline builds without breaking services that don't need them?**

 **Answer:** Attach background `services:` at the **job level** inside your central reusable workflow:
 
 YAML
 
 ```
 jobs:
   build-and-publish:
     runs-on: ubuntu-latest
     services:
       mysql:
         image: mysql:8.0
         env:
           MYSQL_ROOT_PASSWORD: root
           MYSQL_DATABASE: testdb
         ports:
           - 3306:3306
       redis:
         image: redis:alpine
         ports:
           - 6379:6379
 
 ```
 
 -   **Performance Note:** Non-database microservices will safely ignore ports `3306` and `6379` without issue.
     

### 7. Spring Boot Entity Schema & Table Auto-Creation

#### **Q: Do I need manual `CREATE DATABASE` or `CREATE TABLE` scripts in GitHub Actions before running tests?**

 **Answer:** **No.**
 
 1.  **Database:** Created automatically by the Docker image environment variable (`MYSQL_DATABASE: testdb`).
     
 2.  **Tables:** Spring Data JPA / Hibernate generates all entities on the fly when `src/test/resources/application.properties` specifies:
     
     Properties
     
     ```
     spring.datasource.url=jdbc:mysql://localhost:3306/testdb?useSSL=false
     spring.datasource.username=root
     spring.datasource.password=root
     spring.jpa.hibernate.ddl-auto=create-drop
     
     ```
     
 3.  **Lifecycle:** The database and all generated tables are completely destroyed when the GitHub Actions job finishes running.
 
    
### 8. Resolving Missing Dependency Version in Spring Boot 4.0

**Q: Why did `mvn package` fail with `'dependencies.dependency.version' for org.springframework.boot:spring-boot-starter-aop:jar is missing` when using Spring Boot 4.0.5?**

**Answer:**

In Spring Boot 4.x, `spring-boot-starter-aop` was renamed to `spring-boot-starter-aspectj`. Because `spring-boot-starter-aop` is no longer part of the Spring Boot 4.0 Bill of Materials (`spring-boot-dependencies`), Maven could not resolve its managed version automatically.

**Fix:**

Replace the dependency artifact in `pom.xml`:

XML

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aspectj</artifactId>
</dependency>

```

### 9. Bill of Materials (BOM) & Hierarchy Errors in Maven

**Q: What is a Bill of Materials (BOM), what happens if `<dependencyManagement>` is placed inside `<dependencies>`, and is explicit BOM import needed when using `spring-boot-starter-parent`?**

**Answer:**

-   **BOM (Bill of Materials):** A curated `pom.xml` that defines verified, compatible dependency versions (preventing version mismatch errors like `NoSuchMethodError`).
    
-   **XML Hierarchy Rule:** `<dependencyManagement>` **must** sit at the root level under `<project>` as a sibling to `<dependencies>`. Placing `<dependencyManagement>` inside `<dependencies>` causes Maven parsing errors, ignoring the imported BOM entirely.
    
-   **Parent vs. DependencyManagement:** If your `pom.xml` defines `spring-boot-starter-parent` in its `<parent>` section, you **do not** need a separate `<dependencyManagement>` block to import `spring-boot-dependencies`. The parent POM imports all version rules automatically.
    

### 10. Docker Build Failure: `JAR file not found in target/`

**Q: Why did Docker fail with `COPY target/*.jar: not found` during the GitHub Actions build, and how do we fix it cleanly across all microservices?**

**Answer:**

-   **Root Cause:** The `docker build` step was executing before Maven compiled the fat JAR, or the hardcoded JAR filename in the `Dockerfile` didn't match the version output from Maven.
    
-   **Fix:**
    
    1.  Ensure `mvn clean deploy` (or `mvn clean package`) runs **before** the `docker/build-push-action` step in your pipeline.
        
    2.  Use a generic wildcard copy in your `Dockerfile` so it works for all microservices without updating hardcoded versions:
        

Dockerfile

```
FROM eclipse-temurin:21-jdk-jammy  
COPY target/*.jar app.jar  
ENTRYPOINT ["java","-jar","/app.jar"]

```

### 11. Role of `<name>` and `<finalName>` in `pom.xml`

**Q: Is the `<name>` tag mandatory in `pom.xml`? How does `<finalName>` help with Docker builds?**

**Answer:**

-   **`<name>`:** Not mandatory. It is purely display metadata for IDEs and reports. If omitted, Maven defaults to using `<artifactId>`.
    
-   **`<finalName>`:** Optional, but useful if you want to explicitly control the output filename in `target/` (e.g., `<finalName>${project.artifactId}</finalName>`), producing `spring-order-service.jar` instead of `spring-order-service-0.0.1-SNAPSHOT.jar`.
    

### 12. Maven Lifecycle & `mvn clean deploy`

**Q: Does running `mvn clean deploy` run tests and package the JAR, or do we need `mvn clean package deploy`?**

**Answer:**

You only need `mvn clean deploy`. Maven's build phases are strictly sequential:

$$\text{compile} \longrightarrow \text{test} \longrightarrow \mathbf{\text{package}} \longrightarrow \text{install} \longrightarrow \mathbf{\text{deploy}}$$

Invoking `deploy` automatically runs `test` and `package` beforehand. Specifying both `package` and `deploy` causes Maven to execute packaging logic twice unnecessarily.

### 13. Fix 403 Forbidden Errors from Maven Central in GitHub Actions

**Q: Why did `mvn clean deploy` throw `403 Forbidden` when attempting to fetch `spring-boot-starter-parent` from Maven Central?**

**Answer:**

-   **Root Cause:** Using `actions/setup-java@v4` with `server-id: github` configures `~/.m2/settings.xml` with GitHub authentication tokens. When Maven attempted to download public artifacts from Maven Central, it sent those GitHub authorization headers, which Maven Central rejected with `403 Forbidden`.
    
-   **Fix:** Add `-DsuppressCentralAuth=true` to the Maven command or explicitly map the central repository in `pom.xml`:
    

Bash

```
mvn clean deploy -DsuppressCentralAuth=true

```

### 14. Fixing GitHub Packages Package Deployment Lookups

**Q: Why did `maven-deploy-plugin` fail with `Could not find artifact in github ([https://maven.pkg.github.com/VamshiOrganization/product-service](https://maven.pkg.github.com/VamshiOrganization/product-service))`?**

**Answer:**

-   **Root Causes:**
    
    1.  Missing workflow write permissions for packages (`packages: write`).
        
    2.  Appending `${project.artifactId}` directly to the GitHub Packages repository URL in `<distributionManagement>` can cause resolution lookup failures for newly created repositories.
        
-   **Fix:** Set permissions in the workflow and point `<distributionManagement>` directly to the Organization endpoint:
    

YAML

```
# In .github/workflows/pipeline.yml
permissions:
  contents: read
  packages: write

```

XML

```
<!-- In pom.xml -->
<distributionManagement>
    <repository>
        <id>github</id>
        <name>GitHub Packages</name>
        <url>https://maven.pkg.github.com/VamshiOrganization</url>
    </repository>
</distributionManagement>

```

### 15. Timestamped Naming for `-SNAPSHOT` Artifacts in GitHub Packages

**Q: Why does GitHub Packages rename `product-service-1.0.0-SNAPSHOT.jar` to `product-service-1.0.0-20260722.181312-1.jar` upon upload?**

**Answer:**

-   **Standard Maven SNAPSHOT Behavior:** Local builds in `target/` overwrite files using `-SNAPSHOT.jar`. Remote registries (like GitHub Packages) append a UTC timestamp and build counter to maintain immutable snapshot history and allow builds to be tracked.
    
-   **Resolution:** You still declare `<version>1.0.0-SNAPSHOT</version>` in dependent projects—Maven uses `maven-metadata.xml` behind the scenes to fetch the latest timestamped artifact automatically.


### 16. Maven Local Build (`target/`) vs. GitHub Packages Deployment

how **Maven Build (Local Packaging)** and **GitHub Packages (Remote Registry Deployment)** interact, focusing on why snapshots behave differently in each environment and how they stay synchronized.

**Q: How does Maven handle SNAPSHOT JAR creation locally in `target/` versus remotely in GitHub Packages during `mvn clean deploy`?**

**Answer:**

When you run `mvn clean deploy`, Maven executes two distinct phases sequentially for the target artifact:

```
[Local Phase]                               [Remote Phase]
mvn package                                 mvn deploy
  │                                           │
  ▼                                           ▼
Generates:                                  Uploads to GitHub Packages as:
target/product-service-1.0.0-SNAPSHOT.jar   product-service-1.0.0-20260722.181312-1.jar
(Single, local overwriting file)            (Unique, timestamped immutable build)
```


### Local Build (`target/`) vs GitHub Packages (Remote Registry)

| **Aspect** | **Local Build (`target/`)** | **GitHub Packages (Remote Registry)** |
|---|---|---|
| **Filename Format** | `product-service-1.0.0-SNAPSHOT.jar` | `product-service-1.0.0-YYYYMMDD.HHMMSS-B.jar` |
| **Storage Behavior** | **Mutable / Overwrites:** Every `mvn package` replaces the previous JAR in the `target/` directory. | **Immutable / Appends:** Every `mvn deploy` publishes a new timestamped artifact and updates `maven-metadata.xml`. |
| **Primary Consumer** | **Docker Build Context:** The `Dockerfile` copies the JAR directly from the local `target/` directory during `docker build`. | **Downstream Microservices:** Other applications retrieve the published JAR as a Maven dependency from **GitHub Packages**. |

#### Why the Distinction Matters:

1.  **For Docker Containerization (`COPY target/*.jar app.jar`):**
    
    Docker context runs on the GitHub Runner _locally_ before container creation. It copies `target/product-service-1.0.0-SNAPSHOT.jar` directly from the local workspace **before** or **after** `deploy` runs. The timestamped remote URL on GitHub Packages has zero impact on the Docker build.
    
2.  **For Dependency Resolution (`maven-metadata.xml`):**
    
    When another microservice lists your library as `<version>1.0.0-SNAPSHOT</version>`, Maven queries the `maven-metadata.xml` file hosted inside `[https://maven.pkg.github.com/VamshiOrganization](https://maven.pkg.github.com/VamshiOrganization)`. It reads the latest timestamp (`20260722.181312-1`) and pulls that exact file transparently.
