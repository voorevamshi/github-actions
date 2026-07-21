

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
 
    

