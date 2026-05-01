
# CICD-DEMO

This project aims to be the basic skeleton to apply continuous integration and continuous delivery.

## Topology

CICD Demo uses some kubernetes primitives to deploy:

* Deployment
* Services
* Ingress ( with TLS )

```bash
     internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
   --|-----|--
   [   Pods   ]

```

This project includes:

* Spring Boot java app
* Jenkinsfile integration to run pipelines
* Dockerfile containing the base image to run java apps
* Makefile and docker-compose to make the pipeline steps much simpler
* Kubernetes deployment file demonstrating how to deploy this app in a simple Kubernetes cluster

## Pipeline CI/CD (Jenkins)

Este proyecto se construye y despliega con un pipeline declarativo definido en
[`Jenkinsfile`](./Jenkinsfile). Jenkins corre en un contenedor local con acceso
al socket Docker del host, y se apoya en SonarQube y Trivy como puertas de
calidad antes de desplegar.

### Topología local

```
  ┌─────────────┐        ┌──────────────┐        ┌──────────────┐
  │   Jenkins   │──Sonar▶│  SonarQube   │        │     Trivy    │
  │ (contenedor)│        │ (contenedor) │        │  (CLI local) │
  └──────┬──────┘        └──────────────┘        └──────────────┘
         │ docker.sock
         ▼
   ┌─────────────┐
   │   mi-app    │   docker run -d -p 80:8080
   └─────────────┘
```

### Flujo del Pipeline

| # | Stage | Acción | Falla si... |
|---|---|---|---|
| 1 | `Checkout` | Clona el repo desde el SCM configurado en el job | el clon falla |
| 2 | `Build` | `./mvnw -DskipTests clean package` | la compilación o el packaging fallan |
| 3 | `Test` | `./mvnw test` y publica reportes JUnit | algún test unitario falla |
| 4 | `Docker Build` | `docker build -t mi-app:latest .` | el build de imagen falla |
| 5 | `Static Analysis (SonarQube)` | `mvn sonar:sonar` con `withSonarQubeEnv` | el escaneo no puede subir resultados |
| 6 | `Quality Gate` | `waitForQualityGate abortPipeline: true` | SonarQube reporta Quality Gate **fallida** (incluye Security Hotspots no revisados) |
| 7 | `Container Security Scan (Trivy)` | `trivy image --exit-code 1 --severity CRITICAL` | la imagen tiene **alguna** vulnerabilidad CRITICAL |
| 8 | `Deploy` (solo `main`/`master`) | `docker run -d -p 80:8080 mi-app:latest` | no logra levantar el contenedor |

### Manejo de errores y limpieza

El bloque `post` deja el entorno consistente en cualquier resultado:

- `post.success`: archiva el `.jar` en `target/` y registra el despliegue.
- `post.failure`: imprime el stage que rompió y elimina el contenedor `mi-app`
  si quedó a medio desplegar (`docker rm -f mi-app || true`).
- `post.always`: limpia el workspace de Jenkins (`cleanWs()`).

### Pre-requisitos del agente Jenkins

Para que el pipeline funcione end-to-end el contenedor Jenkins necesita:

- Acceso al socket Docker del host
  (`-v /var/run/docker.sock:/var/run/docker.sock`).
- Docker CLI instalado dentro del contenedor.
- Trivy CLI instalado dentro del contenedor.
- Conectividad de red al contenedor `sonarqube` (red Docker compartida).
- Una credencial Jenkins llamada `SonarQube` (token generado en SonarQube)
  registrada vía *Manage Jenkins → System → SonarQube servers*.

### Cómo configurar el Job en Jenkins

1. *New Item* → tipo **Pipeline** → nombre `cicd-demo`.
2. En la sección *Pipeline*:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `https://github.com/ItsJuanda17/cicd-demo.git`
   - Branch Specifier: `*/master`
   - Script Path: `Jenkinsfile`
3. *Save* → *Build Now*.

### Pipelines legados

El pipeline original (con despliegues a Kubernetes y push a registry) se
conserva como referencia en [`Jenkinsfile.original`](./Jenkinsfile.original).

How to run the app:

```make
make
```

## Testing

Unit tests and integrations tests are separated using [JUnit Categories][].

[JUnit Categories]: https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit.html

### Unit Tests

```java
mvn test -Dgroups=UnitTest
```

Or using Docker:

```bash
make build
```

### Integration Tests

```java
mvn integration-test -Dgroups=IntegrationTests
```

Or using Docker:

```bash
make integrationTest
```

### System Tests

System tests run with Selenium using docker-compose to run a [Selenium standalone container][] with Chrome.

[Selenium standalone container]: https://github.com/SeleniumHQ/docker-selenium

Using Docker:

* If you are running locally, make sure the `$APP_URL` is populated and points to a valid instance of your application. This variable is populated automatically in Jenkins.

```bash
APP_URL=http://dev-cicd-demo-master.anzcd.internal/ make systemTest
```