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

Example layered, multi-stage Dockerfile (small runtime image, cache-friendly):

```dockerfile
# Extract Boot layers from the fat JAR
FROM eclipse-temurin:17-jre-jammy AS builder
WORKDIR /app
ARG JAR_FILE=target/app.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Final image: JRE only, non-root, most-stable layers first
FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./
USER 1000
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
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

Boot uses `JarLauncher` with a dedicated launched class loader to load nested JARs (the loader was reworked in Boot 3.2). Native image builds use AOT compilation and adapt Spring context initialization for GraalVM.

## Interview Q&A

**Q: How does an executable Spring Boot "fat JAR" work?**
A: The build plugin nests application classes and dependency JARs inside one archive with a `JarLauncher` main class and a custom class loader that reads the nested JARs directly, so `java -jar app.jar` runs without unpacking.

**Q: Why do layered JARs improve Docker builds?**
A: They split the JAR into layers that change at different rates (dependencies, spring-boot-loader, snapshot-dependencies, application). Copying stable layers first lets Docker cache them, so a code change only rebuilds the small application layer.

**Q: How do you configure a Boot app per environment without rebuilding?**
A: Externalize config — environment variables (relaxed binding), `SPRING_PROFILES_ACTIVE`, mounted config files, or `spring.config.import` for config server/Vault — so the same image runs everywhere.

**Q: What is the difference between liveness and readiness in Kubernetes?**
A: Liveness failure restarts the pod; readiness failure removes it from the Service endpoints until it recovers. Boot maps these to Actuator health groups (`probes.enabled=true`).

**Q: What do blue-green and canary deployments give you?**
A: Blue-green keeps two environments and switches traffic atomically for instant rollback; canary shifts a small percentage of traffic to the new version first to limit blast radius.

**Q: What does a GraalVM native image change about a Boot app?**
A: AOT processing generates ahead-of-time bean/proxy metadata and reflection hints so the app compiles to a native binary with very fast startup and low memory — at the cost of longer builds and some reflection/dynamic-feature constraints.

## Interview Notes

* Explain how Boot packages an executable JAR and why layered JARs improve Docker build performance.
* Describe how to expose configuration and secrets in cloud environments.
* Know how to use Actuator readiness and liveness probes in Kubernetes.
