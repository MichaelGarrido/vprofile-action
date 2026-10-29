# VProfile application and deployment examples

A Java web application for account registration, sign-in, profile management, and demonstrations of caching, messaging, and search. It packages a Spring MVC application as a WAR and includes a Tomcat container image recipe, Kubernetes workload examples, a Jenkins pipeline, and Ansible deployment playbooks.

This is a fork of [hkhcoder/vprofile-action](https://github.com/hkhcoder/vprofile-action). The application and deployment examples originate from that upstream project. This fork documents the inherited implementation and the work still needed to operate it reliably.

## Stack and repository map

| Area | Source |
| --- | --- |
| Java application | [`src/main/java`](src/main/java), Spring MVC/Security/Data JPA |
| JSP interface and Spring XML configuration | [`src/main/webapp`](src/main/webapp) |
| Build and tests | [`pom.xml`](pom.xml), [`src/test/java`](src/test/java) |
| Service connection settings and SQL fixtures | [`src/main/resources`](src/main/resources) |
| Container build | [`Dockerfile`](Dockerfile): Java 11 build stage, Tomcat 9 / Java 11 runtime |
| Kubernetes examples | [`kubernetes/vpro-app`](kubernetes/vpro-app): app, MySQL, Memcached, RabbitMQ, and ingress |
| Jenkins artifact pipeline | [`Jenkinsfile`](Jenkinsfile): build, tests, analysis, and Nexus publication |
| Host deployment examples | [`ansible`](ansible): Tomcat installation and WAR deployment |

The Maven compiler targets Java 8 bytecode; the Dockerfile builds and runs with Java 11. The application uses Spring 4.x and an Elasticsearch 5.6.4 transport client. See the setup guide for compatibility and deployment considerations.

## Build locally

Start with JDK 11 and Maven 3 to match the intended container build environment:

```sh
git clone https://github.com/MichaelGarrido/vprofile-action.git
cd vprofile-action
mvn test
mvn package
```

Maven writes `target/vprofile-v2.war`. The repository includes JUnit tests. Dependencies must be available from the configured Maven repositories.

To build the supplied container image:

```sh
docker build -t vprofile-local:dev .
```

The Dockerfile runs `mvn install` and copies the resulting WAR to Tomcat as `ROOT.war`, listening on port `8080`. A usable application also needs configured backing services. Follow the [setup guide](docs/setup.md) before running or deploying it.

## Architecture and deployment status

The Kubernetes examples connect the application to MySQL, Memcached, and RabbitMQ through cluster Services. The source also supports Elasticsearch and a standby cache, but their workloads are not included. AWS VPC and EKS configuration is in the companion [iac-vprofile repository](https://github.com/MichaelGarrido/iac-vprofile).

This checkout has no GitHub Actions application pipeline, Argo CD Application, Helm chart, or container publication step. The Jenkinsfile publishes a WAR and POM to Nexus; it does not deploy Kubernetes. The manifests use pre-existing images and contain configuration and persistence gaps described in the setup guide.

- [Architecture](docs/architecture.md): components, request paths, and manifest coverage.
- [Setup and limitations](docs/setup.md): configuration, build prerequisites, database initialization, and deployment blockers.
- [Pipeline guide](docs/pipeline.md): Jenkins stages, external dependencies, and the separate Ansible path.
