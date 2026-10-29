# Build and deployment paths

[Back to README](../README.md) · [Architecture](architecture.md) · [Setup](setup.md)

The repository contains three separate mechanisms: a Jenkins WAR publishing pipeline, a Dockerfile, and Ansible/Kubernetes deployment examples. No checked-in automation connects all of them into an end-to-end delivery pipeline.

## Jenkins pipeline

[`Jenkinsfile`](../Jenkinsfile) declares the following stages:

| Stage | Command or action |
| --- | --- |
| Build | `mvn clean install -DskipTests`, then archive `target/*.war` |
| Unit test | `mvn test` |
| Integration test | `mvn verify -DskipUnitTests` |
| Checkstyle | `mvn checkstyle:checkstyle` |
| SonarQube analysis | Configured Sonar scanner, followed by a quality-gate wait with a ten-minute timeout |
| Nexus publication | Upload WAR and POM using the Jenkins build ID as the artifact version |

The stage labeled “Integration test” does not establish a distinct integration test suite: the POM has no Failsafe configuration and does not define behavior for `skipUnitTests`. Review Maven's effective lifecycle before interpreting this stage as independent integration coverage.

The pipeline expects:

- Maven and a compatible JDK on the selected Jenkins agent. Its `tools` block is commented out.
- A scanner tool named `sonarscanner4`, a SonarQube installation named `sonar-pro`, and quality-gate integration.
- Jenkins support for `readMavenPom`, `findFiles`, and `nexusArtifactUploader`.
- A reachable Nexus server/repository and a Jenkins credential entry named `nexuslogin`.

The Nexus location and repository are hardcoded environment settings in the Jenkinsfile, and the publication protocol is HTTP. They are environment-specific examples. Change them before operating a new pipeline; do not copy credentials into source control.

There is no container registry push, Kubernetes deployment, Ansible invocation, or branch guard around Nexus publication in this Jenkinsfile. Jenkins triggers and job configuration live outside this repository. A repository push can invoke a separately configured Jenkins job, so check that configuration before publishing changes.

## Container build

[`Dockerfile`](../Dockerfile) uses two stages:

1. `openjdk:11` installs Maven and runs `mvn install` against the copied checkout.
2. `tomcat:9-jre11` receives `target/vprofile-v2.war` as `ROOT.war` and starts Tomcat on port `8080`.

The repository does not supply a registry destination, image tag convention, digest promotion, or an automated step that updates Kubernetes to the new image. Application properties are included in the WAR; the manifest Secret is not automatically injected into them.

## Ansible deployment

[`ansible/site.yml`](../ansible/site.yml) imports the Tomcat setup and WAR deployment playbooks. They target `appsrvgrp`, install Java 8/Tomcat 8 on older CentOS/Ubuntu variants, download an artifact from Nexus, and replace Tomcat's root application with backup/restore tasks.

The WAR download expects externally supplied variables including `USER`, `PASS`, `nexusip`, `reponame`, `groupid`, `time`, `build`, and `vprofile_version`. No inventory or complete variable set is supplied. The download embeds credentials in an HTTP URL, the application configuration template task is commented out, and host-key checking is disabled in the supplied Ansible configuration. Treat these as legacy examples requiring adaptation and review.

## GitHub Actions and GitOps status

The `main` tree inspected at commit `29d31a9` contains no `.github/workflows` files, Argo CD Application, or Helm chart. The companion [iac-vprofile](https://github.com/MichaelGarrido/iac-vprofile) repository has a Terraform workflow on its `stage` branch; that workflow provisions infrastructure and does not build or deploy this application.
