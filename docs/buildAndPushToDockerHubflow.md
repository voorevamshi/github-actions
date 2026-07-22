developer commit to Maven build, Docker packaging, and deployment to GitHub Packages.

```
       +-------------------------------------------------------+
       |                  Developer Commit / Push              |
       +-------------------------------------------------------+
                                   |
                                   v
       +-------------------------------------------------------+
       |                 GitHub Actions Runner                 |
       |  (actions/checkout & setup-java: JDK 21 Temurin)       |
       +-------------------------------------------------------+
                                   |
                                   v
       +-------------------------------------------------------+
       |                   mvn clean deploy                    |
       |       (-DsuppressCentralAuth=true & GITHUB_TOKEN)      |
       +-------------------------------------------------------+
                |                                      |
       (1. Build & Test)                      (2. Publish Artifact)
                |                                      |
                v                                      v
  +---------------------------+          +---------------------------+
  |    Local Maven Target     |          |      GitHub Packages      |
  |                           |          |                           |
  | Generates fat executable: |          | Uploads timestamped build:|
  | target/*.jar              |          | product-service-1.0.0-    |
  |                           |          | 20260722.181312-1.jar     |
  +---------------------------+          +---------------------------+
                |
                | (3. Docker Context reads target/*.jar)
                v
       +-------------------------------------------------------+
       |             docker/build-push-action                 |
       |                                                       |
       | Runs Dockerfile:                                      |
       |   FROM eclipse-temurin:21-jdk-jammy                   |
       |   COPY target/*.jar app.jar                           |
       |   ENTRYPOINT ["java", "-jar", "/app.jar"]             |
       +-------------------------------------------------------+
                                   |
                                   v
       +-------------------------------------------------------+
       |                      Docker Hub                       |
       |             vamshivoore/product-service:latest        |
       +-------------------------------------------------------+

```

### Detailed Breakdown of the Flow

#### 1. Pipeline Authorization & Setup

-   **`actions/setup-java`** injects GitHub credentials into `~/.m2/settings.xml` using `server-id: github`.
    
-   **Central Auth Suppression:** `-DsuppressCentralAuth=true` prevents Maven from attaching GitHub authentication headers when downloading public parent POMs (like `spring-boot-starter-parent`) from Maven Central, avoiding **403 Forbidden** errors.
    

#### 2. Single Maven Build Step (`mvn clean deploy`)

Instead of splitting commands into `mvn package` and `mvn deploy`, running `mvn clean deploy` handles the entire sequential lifecycle automatically:

$$\text{clean} \longrightarrow \text{compile} \longrightarrow \text{test} \longrightarrow \mathbf{\text{package}} \longrightarrow \mathbf{\text{deploy}}$$

-   **Local Target Output (`target/*.jar`):** Creates the executable JAR inside the workspace (`target/product-service-1.0.0-SNAPSHOT.jar`).
    
-   **Remote Registry Upload:** Converts the local `-SNAPSHOT.jar` name into a timestamped immutable artifact (`product-service-1.0.0-20260722.181312-1.jar`) and updates `maven-metadata.xml` on GitHub Packages.
    

#### 3. Containerization Step (`docker/build-push-action`)

-   **Context Resolution:** Docker reads directly from the local workspace's `target/` directory generated during the `mvn clean deploy` phase.
    
-   **Generic Copy:** Using `COPY target/*.jar app.jar` keeps the `Dockerfile` version-agnostic across all microservices, eliminating the need to update hardcoded JAR names when versions change.
    
-   **Registry Push:** Tags and pushes the container image to Docker Hub (`vamshivoore/<repository-name>:latest`).
