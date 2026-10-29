# Setup and known limitations

[Back to README](../README.md) · [Architecture](architecture.md) · [Pipeline](pipeline.md)

This guide covers building the application and preparing its backing services and deployment examples.

## Build prerequisites

- JDK 11 and Maven 3 match the intended Docker build environment. The POM targets Java 8 bytecode.
- Docker is needed only for the container build path.
- Running the application requires MySQL and the configured backing services. The inherited README specified MySQL 8; the POM pins Connector/J `8.0.33` while the Hibernate configuration uses `MySQL5Dialect`, so validate database behavior against your selected server.
- Kubernetes and an NGINX ingress controller are required only for the supplied manifest path. Neither is installed by this repository.

From the repository root:

```sh
mvn test
mvn package
```

The packaged artifact is `target/vprofile-v2.war`. The Docker build performs its own Maven installation phase and deploys that artifact as `ROOT.war` under Tomcat 9. A Java unit test run does not verify MySQL, queue, cache, search, ingress, or Kubernetes behavior.

## Application configuration

[`application.properties`](../src/main/resources/application.properties) is loaded from the application's classpath and packaged into the WAR. Review it before building for an environment. No external configuration loader or automatic mapping from Kubernetes Secrets to these properties is supplied.

| Property group | Current expectation | Preparation required |
| --- | --- | --- |
| `jdbc.*` | MySQL database `accounts`, host `vprodb`, port `3306` | Set an environment-specific application account and initialize the schema |
| `memcached.active.*` | `vprocache01:11211` | Provide a reachable cache |
| `memcached.standBy.*` | `vprocache02:11211` | Supply a standby service or revise the configuration/behavior |
| `rabbitmq.*` | Host `vpromq01`; the checked-in port is `15672` | Reconcile the application, Service, and broker's AMQP port |
| `elasticsearch.*` | `vprosearch01:9300` and a named cluster | Supply a compatible service or revise the search integration |

The tracked properties, Ansible template, and Secret manifest contain sample credentials. Base64 encoding in a Kubernetes Secret does not make a committed value confidential. Replace examples in a private environment, keep real credentials out of commits, and rotate any value that has been used outside the lab. The ignore rules do not protect credentials inside already-tracked source files.

## Database initialization

The inherited setup uses [`src/main/resources/db_backup.sql`](../src/main/resources/db_backup.sql), which contains table definitions and seeded records for `role`, `user`, and `user_role`. A second, different fixture exists as [`accountsdb.sql`](../src/main/resources/accountsdb.sql); do not assume the two are interchangeable.

Prepare an empty `accounts` database and an appropriate account before importing the selected fixture:

```sh
mysql -h YOUR_DB_HOST -u YOUR_DB_USER -p accounts < src/main/resources/db_backup.sql
```

Both SQL files contain `DROP TABLE IF EXISTS` statements. Import only into an intended disposable or backed-up database after reviewing the file. The repository supplies no migration framework or automated local service stack. The database container image is external to this repository; its initialization behavior must be checked separately.

## Before deploying the Kubernetes examples

Resolve these source-defined gaps before treating the manifests as a working environment:

1. **RabbitMQ ports are inconsistent.** The [Service](../kubernetes/vpro-app/rmq-CIP-service.yml) exposes `5672` and targets a named port defined as `15672` in the [Deployment](../kubernetes/vpro-app/rmq-dep.yml). The application also configures `15672`. Align the application, Service, and actual broker listener; a `containerPort` declaration does not configure the broker.
2. **Database storage is ephemeral.** The [database Deployment](../kubernetes/vpro-app/vprodbdep.yml) has no persistent volume claim or volume mount. Replacing its pod can lose local database state. The application's uploaded images also have no persistent mount.
3. **Service readiness is not checked.** Deployments have no readiness/liveness probes or resource requests/limits. App init containers check DNS only and do not wait for RabbitMQ.
4. **Several dependencies are external.** There is no Elasticsearch or standby Memcached workload, ingress controller installation, or cluster provisioning in this repository.
5. **Image provenance is not tied to this checkout.** The app manifest references a pre-existing image rather than a digest built from this source. Pin and verify intended images before relying on reproducible deployment behavior.
6. **Configuration needs environment-specific handling.** Replace the ingress hostname, arrange DNS, configure TLS if needed, and deliver application connection settings consistently with the database and broker credentials.
7. **Application hardening remains separate work.** The Spring Security configuration explicitly disables CSRF. Dependency versions, access rules, upload handling, and data/log handling require review before exposing the application beyond a controlled lab.

After resolving these gaps, validate the manifest set and verify each service, rollout, and user flow in the intended environment.

## Ansible path

The [Ansible playbooks](../ansible) form a separate host-based deployment path for Java 8 and Tomcat 8. They expect an inventory group named `appsrvgrp`, privilege escalation, Nexus access, and additional variables not defined in the repository. The application-properties template task is commented out, and `ansible.cfg` disables host-key checking. See the [pipeline guide](pipeline.md) before adapting this legacy path.
