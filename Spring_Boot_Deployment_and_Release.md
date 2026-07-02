# Spring Boot Deployment and Release

## Packaging Options

Spring Boot supports executable JARs and WARs.

### Executable JAR

Boot packages dependencies inside a single fat JAR using `JarLauncher` or `PropertiesLauncher`. The build plugins create nested archives for application classes, dependencies, and resources.

### Layered JARs

Layered JAR support separates dependencies, resources, and application classes into distinct layers to improve Docker caching and reduce rebuild time.

### WAR packaging

For servlet containers, Boot can build a WAR and deploy it to external Tomcat, Jetty, or Undertow. A `SpringBootServletInitializer` subclass is required.

## Docker and Containerization

Use a lightweight base image such as `eclipse-temurin` or `distroless`.

Example Dockerfile:

```dockerfile
FROM eclipse-temurin:17-jdk-jammy
ARG JAR_FILE=target/app.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### Best practices

* run as a non-root user
* use layered JARs for build cache efficiency
* externalize configuration via environment variables
* minimize image size and disable debug logging in production

## Cloud Deployment

### PaaS and Kubernetes

Spring Boot apps run well on Heroku, Cloud Foundry, AWS Elastic Beanstalk, Azure App Service, and Kubernetes.

### Configuration in cloud

Use environment variables, config maps, and secrets. Boot resolves environment variables with relaxed binding.

## CI/CD and Release Management

### Build pipelines

Use Maven or Gradle to compile, test, and package artifacts. Bake container images and run integration tests in CI.

### Versioning

Adopt semantic versioning and automate artifact tagging in CI/CD.

### Deployment strategies

Blue-green and canary deployments reduce risk and support safe rollbacks.

## Runtime Configuration

### JVM tuning

Configure `-Xms`, `-Xmx`, heap settings, and GC options based on production workload. Monitor using JFR and APM.

### Spring Boot runtime properties

Externalize runtime settings with environment variables or config files. Use `SPRING_PROFILES_ACTIVE`, `JAVA_OPTS`, and `SPRING_APPLICATION_JSON`.

## Health and Readiness

Actuator readiness and liveness probes are essential for Kubernetes and container orchestrators.

* `management.endpoint.health.probes.enabled=true`
* use readiness and liveness groups to separate startup checks from runtime health

## Internal Startup Behavior

Boot uses `JarLauncher` with `LaunchedURLClassLoader` to load nested JARs. Native image builds use AOT compilation and adapt spring context initialization for GraalVM.

## Interview Notes

* Explain how Boot packages an executable JAR and why layered JARs improve Docker build performance.
* Describe how to expose configuration and secrets in cloud environments.
* Know how to use Actuator readiness and liveness probes in Kubernetes.
