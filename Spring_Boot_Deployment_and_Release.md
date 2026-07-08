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

## Repackaging Internals

The `spring-boot-maven-plugin` `repackage` goal turns the plain jar into an **executable jar**: your classes go under `BOOT-INF/classes`, dependencies under `BOOT-INF/lib`, and the manifest sets `Main-Class` to a Boot launcher with `Start-Class` pointing at your `main`. The launcher's class loader reads the nested jars directly. Boot 3.2 rewrote the loader (new `org.springframework.boot.loader.launch` package).

## Layered Jars and jarmode

A `layers.idx` groups contents into layers (dependencies, spring-boot-loader, snapshot-dependencies, application). Extract them with `java -Djarmode=layertools -jar app.jar extract` (Boot 3.3 adds `-Djarmode=tools`), then copy stable layers first in the Dockerfile for maximal cache reuse.

## Cloud Native Buildpacks

`./mvnw spring-boot:build-image` builds an OCI image with **Paketo buildpacks** — no Dockerfile — producing reproducible, layered images with a JVM memory calculator and patchable base images. It is the lowest-friction path to a production-grade container.

## Container Best Practices

Use a slim **JRE/distroless** base, run as **non-root**, prefer a read-only filesystem, and set resource **requests/limits**. Modern JVMs are container-aware (`-XX:+UseContainerSupport`, `MaxRAMPercentage`); ensure clean PID 1 signal handling so shutdown works.

## Kubernetes Deployment

Map Actuator **liveness/readiness/startup** probes to `health/liveness` and `health/readiness`. For zero-downtime rollouts combine `server.shutdown=graceful`, a pod `terminationGracePeriodSeconds`, and a `preStop` hook. Mount **ConfigMaps/Secrets** and import them via `spring.config.import`; scale with an **HPA** on CPU or custom metrics; tune rolling-update `maxSurge`/`maxUnavailable`.

## Native Image

GraalVM native images (via buildpacks or the native-maven-plugin) use Boot **AOT** to precompute beans and **reflection/resource/proxy hints** (`RuntimeHintsRegistrar`, `@RegisterReflectionForBinding`). The payoff is tens-of-milliseconds startup and low memory; the cost is longer builds and constraints on dynamic/reflective features.

## CI/CD and Release Strategies

A pipeline builds -> tests -> scans -> images -> deploys, with semantic artifact versioning. Roll out with **rolling** (K8s default), **blue-green** (switch traffic atomically for instant rollback), or **canary** (progressive traffic shift) deployments, gate on health checks, and use feature flags to decouple deploy from release.

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

**Q: What are Cloud Native Buildpacks and why use them over a Dockerfile?**
A: `spring-boot:build-image` uses Paketo buildpacks to produce a reproducible, layered OCI image with a JVM memory calculator and a patchable base image — no Dockerfile to maintain.

**Q: What container best practices matter for a JVM app?**
A: Use a slim JRE/distroless base, run as non-root, set resource requests/limits, rely on container-aware JVM flags (`MaxRAMPercentage`), and ensure clean PID 1 signal handling for shutdown.

**Q: How do you achieve a zero-downtime rollout in Kubernetes?**
A: Combine readiness probes (mapped to Actuator), `server.shutdown=graceful`, a pod `terminationGracePeriodSeconds`/`preStop` hook, and a rolling update with sensible `maxSurge`/`maxUnavailable`.

**Q: How is the JVM made container-aware for memory?**
A: Modern JVMs honor cgroup limits by default (`-XX:+UseContainerSupport`); size the heap relative to the container with `-XX:MaxRAMPercentage` instead of a fixed `-Xmx`.

**Q: What changed in the Boot 3.2 jar launcher?**
A: The nested-jar loader was rewritten and moved to the `org.springframework.boot.loader.launch` package (e.g. `JarLauncher`), which matters for custom `ENTRYPOINT`s that reference the launcher class.

## Interview Notes

* Explain how Boot packages an executable JAR and why layered JARs improve Docker build performance.
* Describe how to expose configuration and secrets in cloud environments.
* Know how to use Actuator readiness and liveness probes in Kubernetes.
