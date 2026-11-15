# Spring PetClinic Microservices — DevOps and Containerization

[![Build Status](https://github.com/DharaniVellaiyan1441/spring-petclinic/actions/workflows/maven-build.yml/badge.svg)](https://github.com/DharaniVellaiyan1441/spring-petclinic/actions/workflows/maven-build.yml)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

This repository contains a distributed Spring PetClinic application built with Spring Boot and Spring Cloud. It
includes customer, veterinary, visits, and GenAI services, plus configuration, discovery, gateway, monitoring,
and tracing components. This guide describes the repository's Docker setup, local development options, and
CI/CD workflows.

## Architecture and container inventory

The application services communicate over the `petclinic-net` Docker Compose network. The API Gateway is the
browser-facing entry point. Eureka provides service discovery, and Config Server reads application settings
from the mounted `config-repo` directory.

| Compose service | Purpose | Host port |
|---|---|---:|
| `api-gateway` | Routes browser requests to application services | 8080 |
| `config-server` | Serves centralized application configuration | 8888 |
| `discovery-server` | Eureka service registry | 8761 |
| `customers-service` | Manages owners and pets | 8081 |
| `visits-service` | Manages pet visit records | 8082 |
| `vets-service` | Manages veterinarians and specialties | 8083 |
| `genai-service` | Provides the chatbot service | 8084 |
| `admin-server` | Spring Boot Admin dashboard | 9090 |
| `tracing-server` | Zipkin distributed tracing | 9411 |
| `prometheus-server` | Collects application metrics | 9091 |
| `grafana-server` | Displays metrics dashboards | 3030 |

Compose health checks and `depends_on` conditions help ensure Config Server and Discovery Server are ready
before dependent applications start. Allow time for Eureka registration after startup.

## Build and run with Docker Compose

### Prerequisites

- Docker Desktop with Docker Compose v2.
- Java 21 (the version required by the Maven build).
- MySQL 8 running on the host if using the MySQL profile.

### Build images

From the repository root, build and install the application images with the Maven wrapper:

```powershell
.\mvnw.cmd clean install -P buildDocker
```

The Compose file uses the resulting local `spring-petclinic-*:v1` images. Build the images again after changing
application code or image configuration.

### Configure host MySQL

MySQL is **not** a Docker Compose service. It runs on your host and retains its data independently of the
application containers. The Customers, Visits, and Vets containers connect to `host.docker.internal:3306`;
Compose maps that host name to the Docker host gateway.

Create the `petclinic` database and an application account permitted to connect from Docker. Compose defaults
to the development credentials `petclinic` / `petclinic`. Use your own account and set its matching values
before running Compose if those defaults do not match your local MySQL:

```powershell
$env:MYSQL_HOST = "host.docker.internal"
$env:MYSQL_PORT = "3306"
$env:MYSQL_USER = "your-mysql-user"
$env:MYSQL_PASSWORD = "your-mysql-password"
```

The database schema and sample data are not created by the current Compose configuration. On a fresh database,
run each service's MySQL `schema.sql` and `data.sql` once, in this order:

1. [Customers schema](spring-petclinic-customers-service/src/main/resources/db/mysql/schema.sql) and
   [data](spring-petclinic-customers-service/src/main/resources/db/mysql/data.sql)
2. [Vets schema](spring-petclinic-vets-service/src/main/resources/db/mysql/schema.sql) and
   [data](spring-petclinic-vets-service/src/main/resources/db/mysql/data.sql)
3. [Visits schema](spring-petclinic-visits-service/src/main/resources/db/mysql/schema.sql) and
   [data](spring-petclinic-visits-service/src/main/resources/db/mysql/data.sql)

Run Customers first because Visits data refers to pets created by Customers. Schema and sample data persist in
MySQL; these scripts do not need to be run every time the application starts.

### Start, inspect, and stop

```powershell
docker compose up -d
docker compose ps
```

Open the application at http://localhost:8080. Eureka is at http://localhost:8761; Grafana,
Prometheus, and Zipkin are at ports 3030, 9091, and 9411 respectively.

To inspect logs or stop the app containers:

```powershell
docker compose logs -f vets-service
docker compose down
```

`docker compose down` stops and removes the Compose containers and network; it does not stop or delete the host
MySQL installation or its data.

## Run services locally without Docker

Install Java 21. Run each service in its own terminal from the repository root. Start Config Server first, then
Discovery Server, and then the application services. For example, in PowerShell:

```powershell
$env:GIT_REPO = "$PWD\config-repo"
.\mvnw.cmd -pl spring-petclinic-config-server spring-boot:run
```

In another terminal:

```powershell
.\mvnw.cmd -pl spring-petclinic-discovery-server spring-boot:run
```

After those supporting services are ready, start the services you need:

```powershell
.\mvnw.cmd -pl spring-petclinic-customers-service spring-boot:run
.\mvnw.cmd -pl spring-petclinic-vets-service spring-boot:run
.\mvnw.cmd -pl spring-petclinic-visits-service spring-boot:run
.\mvnw.cmd -pl spring-petclinic-api-gateway spring-boot:run
```

When running a service directly on the host with MySQL, activate the `mysql` profile and configure the
`SPRING_DATASOURCE_*` environment variables in that service's terminal. Use `localhost` as the database host
for a locally running Java process; `host.docker.internal` is for containers.

For a mixed setup, `scripts/run_all.sh` starts Grafana, Prometheus, and Zipkin in Compose and starts the Java
applications as local processes. It requires a Bash-compatible shell and Docker Compose. Build the project
first; application logs and PID files are written under `target/`. Use `scripts/stop_all.sh` to stop the apps
and infrastructure started by that helper.

## CI/CD

This repository has two automation workflows:

### GitHub Actions

The [Maven workflow](.github/workflows/maven-build.yml) runs on pushes to `main` and pull requests targeting
`main`. It checks out the code, sets up Temurin Java 21 with Maven dependency caching, and runs
`mvn -B package --file pom.xml` on Ubuntu. This is the repository's build and test check for pull requests.

### Jenkins

The root [Jenkinsfile](Jenkinsfile) defines the image publication and deployment pipeline. Create a Jenkins
Pipeline or Multibranch Pipeline job for this repository. The Jenkinsfile uses Windows `bat` commands, so its
agent must be Windows and have Docker Compose available.

Configure Jenkins with:

- A JDK 21 tool named exactly `JDK-21`.
- Docker and the `docker compose` command available to the agent.
- A Docker Hub username/password credential with ID `dockerhub-creds`.
- For deployment from `main`, host MySQL configured with the `petclinic` database and an account usable by
  the containers. Set `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, and `MYSQL_PASSWORD` in the agent environment
  when the defaults do not fit. Jenkins does not start a MySQL container.

The Jenkins pipeline checks out the branch, packages Config and Discovery Server, builds Gateway and Admin
Server in parallel, and then builds the four application services in parallel. It then builds the Docker
images. These Maven package and image-build commands include `-DskipTests`, so Jenkins itself skips tests.
The `post` section publishes Surefire XML reports if any are present.

For `main`, `develop`, and `release/*`, Jenkins logs in to Docker Hub using the configured credential and pushes
branch/build-number and `latest` tags. For `main` only, it runs `docker compose down` followed by
`docker compose up -d --remove-orphans` on the Jenkins agent. The deployment therefore uses that agent's
Docker daemon, local images, environment variables, and host MySQL.

## Monitoring and troubleshooting

- Check container state with `docker compose ps`.
- Follow a service log with `docker compose logs -f <service-name>`.
- Check Eureka at http://localhost:8761 to confirm services have registered.
- Open Prometheus at http://localhost:9091 and Grafana at http://localhost:3030 for metrics.
- Open Zipkin at http://localhost:9411 for traces. A Zipkin connection warning affects trace reporting; it
  does not by itself mean the application service failed.

For Chaos Monkey usage, see [scripts/chaos/README.md](scripts/chaos/README.md).
