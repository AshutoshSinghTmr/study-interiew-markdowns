# Spring Boot Security and Auth

## Spring Security Architecture

Spring Security secures applications through a chain of filters, authentication providers, and a security context.

### Filter chain

`SecurityFilterChain` is the entry point for HTTP requests. Common filters include:

* `SecurityContextPersistenceFilter`
* `UsernamePasswordAuthenticationFilter`
* `BasicAuthenticationFilter`
* `BearerTokenAuthenticationFilter`
* `ExceptionTranslationFilter`
* `FilterSecurityInterceptor`

The chain delegates authentication to `AuthenticationManager` and stores the result in `SecurityContextHolder`.

### Authentication and authorization

Authentication verifies identity; authorization checks access rights.

* `AuthenticationProvider` validates credentials.
* `AuthenticationManager` coordinates providers.
* `GrantedAuthority` represents permissions.
* `AccessDecisionManager` decides access for protected resources.

## Internal mechanics

### SecurityContext

`SecurityContextHolder` stores the active `SecurityContext` in a thread-local by default. `SecurityContextPersistenceFilter` loads and saves it across requests.

### Method security

`@EnableMethodSecurity` (Spring Security 6; it supersedes the now-removed `@EnableGlobalMethodSecurity`) registers security advisors that intercept method calls. An `AuthorizationManager`-based interceptor evaluates `@PreAuthorize`/`@PostAuthorize` expressions using `MethodSecurityExpressionHandler`.

### Password encoding

`DelegatingPasswordEncoder` supports multiple encodings and accepts legacy passwords through prefixes such as `{bcrypt}`.

## JWT and OAuth2

### JWT flow

JWT validation occurs in a filter, which parses the token and converts it into an `Authentication` object. Spring Security auto-configures JWT support for resource servers when `spring-security-oauth2-resource-server` is on the classpath.

### OAuth2 and OIDC

Spring Security supports OAuth2 login, client registration, and resource server validation. Boot auto-configures clients, `JwtDecoder`, and token introspection when configured.

## CSRF and CORS

### CSRF

CSRF protection is enabled by default for state-changing requests. `CsrfFilter` verifies tokens stored in cookies or request attributes.

### CORS

Configure CORS with `CorsConfigurationSource` and `CorsFilter`. Boot provides default integration with Spring MVC and WebFlux.

## Secure endpoints and REST APIs

Use `SecurityFilterChain` to configure URL authorization rules, session management, and stateless API behavior.

### Common patterns

* stateless JWT APIs with `SessionCreationPolicy.STATELESS`
* component-style `SecurityFilterChain` beans (Spring Security 6 removed `WebSecurityConfigurerAdapter`)
* `oauth2ResourceServer(oauth2 -> oauth2.jwt(...))` for resource servers
* `http.authorizeHttpRequests(...)` for URL rules

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }
}
```

For a JWT resource server, point Boot at the issuer or JWK set:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com/realms/app
```

## Identity and session management

Spring Security manages sessions using `SessionManagementConfigurer` and can control concurrency with session fixation and maximum sessions.

## Advanced topics

### Custom authentication providers

Implement `AuthenticationProvider` to authenticate non-standard credentials or integrate external identity systems.

### Security context propagation

For async execution, use `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` or reactive `SecurityContext` propagation in WebFlux.

### Audit and event handling

Spring Security publishes authentication events and can be extended with `AuthenticationEventPublisher` for auditing.

## The Filter Chain, Ordered

`FilterChainProxy` picks the first `SecurityFilterChain` whose `securityMatcher` matches, then runs its ordered filters. A representative order: `SecurityContextHolderFilter` (loads the `SecurityContext`), `HeaderWriterFilter`, `CorsFilter`, `CsrfFilter`, `LogoutFilter`, authentication filters (`UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`), `AnonymousAuthenticationFilter`, `ExceptionTranslationFilter`, and finally `AuthorizationFilter`. Define multiple chains (each with its own `securityMatcher` and `@Order`) to apply different rules to, say, `/api/**` vs the UI.

## Authentication Architecture

`AuthenticationManager` (usually `ProviderManager`) delegates to a list of `AuthenticationProvider`s until one authenticates:

* `DaoAuthenticationProvider` — `UserDetailsService` + `PasswordEncoder`.
* `JwtAuthenticationProvider` / resource-server decoders — validate tokens.

The resulting `Authentication` (principal, authorities, authenticated flag) is stored in the `SecurityContext`. In Security 6 the context is persisted explicitly via a `SecurityContextRepository` and loaded by `SecurityContextHolderFilter` (the old `SecurityContextPersistenceFilter` is gone; saving is explicit).

## Authorization

Security 6 uses `AuthorizationManager` (replacing `AccessDecisionManager`). URL rules come from `authorizeHttpRequests(...)` with request matchers; method rules from SpEL:

```java
@PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.name")
Order get(String ownerId) { ... }
```

Distinguish **roles** (`ROLE_` prefix, `hasRole`) from raw **authorities** (`hasAuthority`). For per-object rules use domain-object security (ACLs). RBAC assigns permissions by role; ABAC decides from attributes (owner, tenant, time) — expressible in SpEL or a custom `AuthorizationManager`.

## OAuth2 and OIDC

Three roles: **resource server** (validates access tokens), **client** (obtains tokens for users), and **authorization server** (Spring Authorization Server). Common grants: **authorization code + PKCE** (web/SPA login) and **client credentials** (machine-to-machine). OIDC adds an **ID token** describing the user. A `JwtDecoder` validates signature (via the issuer's JWK set), issuer, audience, and expiry; **opaque** tokens are validated by calling the provider's introspection endpoint instead.

## Method Security Internals

`@EnableMethodSecurity` registers `AuthorizationManager`-based interceptors (`AuthorizationManagerBeforeMethodInterceptor`, etc.) that evaluate `@PreAuthorize`/`@PostAuthorize`/`@PreFilter`/`@PostFilter`. Because it is proxy-based, the AOP self-invocation caveat applies.

## Sessions, CSRF, CORS, Headers

Choose `SessionCreationPolicy.STATELESS` for token APIs, or session-based auth with concurrency control and session-fixation protection for server-rendered apps. Enable CSRF for cookie-based browser flows (use `CookieCsrfTokenRepository` for SPAs) and disable it for stateless token APIs. Configure CORS (preflight handling) via a `CorsConfigurationSource`, and set security headers (HSTS, CSP, `X-Frame-Options`) through the headers DSL.

## Passwords and Reactive Security

Store passwords with `DelegatingPasswordEncoder` (bcrypt/argon2) and enable `upgradeEncoding` to re-hash on login. In WebFlux, the equivalents are `SecurityWebFilterChain`, `ServerHttpSecurity`, and `ReactiveSecurityContextHolder` (context flows via the Reactor context, not thread-locals).

## Mapping to OWASP

The stack directly mitigates several OWASP Top 10 items: strong authentication and session management (broken auth), method/URL authorization (broken access control), CSRF tokens (CSRF), security headers/CSP (misconfiguration, XSS), and password hashing/TLS (cryptographic failures). Keep dependencies patched and secrets externalized.

## Interview Q&A

**Q: How is Spring Security configured in Spring Security 6 (no `WebSecurityConfigurerAdapter`)?**
A: Declare one or more `SecurityFilterChain` beans that configure `HttpSecurity` with the lambda DSL. Multiple chains can be ordered and matched to different URL patterns.

**Q: How does a request flow through the security filter chain?**
A: `FilterChainProxy` selects the matching `SecurityFilterChain`; authentication filters build an `Authentication`, `AuthenticationManager`/providers validate it and store it in the `SecurityContextHolder`, then `AuthorizationFilter` enforces access, with `ExceptionTranslationFilter` handling failures.

**Q: What is the difference between authentication and authorization?**
A: Authentication establishes *who* the caller is (credentials -> `Authentication`); authorization decides *what* they may do (roles/authorities checked at URLs or methods).

**Q: How do you secure a stateless REST API with JWTs?**
A: Set `SessionCreationPolicy.STATELESS`, disable CSRF for the API, and enable `oauth2ResourceServer().jwt()` with an `issuer-uri`/JWK set so each request's bearer token is validated and turned into an `Authentication`.

**Q: Why is CSRF protection on by default, and when can you disable it?**
A: It protects browser session-cookie flows from forged state-changing requests. It can be disabled for stateless token-based APIs that do not rely on cookies for authentication.

**Q: How does `DelegatingPasswordEncoder` work?**
A: It stores an algorithm-id prefix like `{bcrypt}` with each hash, so it can verify legacy encodings and upgrade the default algorithm without breaking existing passwords.

**Q: What is the difference between a role and an authority?**
A: A role is just an authority conventionally prefixed with `ROLE_`. `hasRole('ADMIN')` checks `ROLE_ADMIN`; `hasAuthority('ORDER_READ')` checks the raw authority string.

**Q: How does JWT validation differ from opaque-token validation?**
A: A JWT is self-contained and validated locally by a `JwtDecoder` (signature via the JWK set, issuer, audience, expiry); an opaque token carries no claims, so the resource server calls the provider's introspection endpoint to validate it.

**Q: When do you use authorization code + PKCE versus client credentials?**
A: Authorization code + PKCE is for user login from web apps/SPAs (no client secret in the browser); client credentials is for machine-to-machine calls with no user.

**Q: How and why would you define multiple `SecurityFilterChain` beans?**
A: Give each a `securityMatcher` and `@Order` so, for example, `/api/**` uses stateless JWT while the UI uses form login and sessions — different rules without conflicting configuration.

**Q: How does `@PreAuthorize` get enforced?**
A: `@EnableMethodSecurity` registers an `AuthorizationManager`-based method interceptor (AOP) that evaluates the SpEL before the method runs; being proxy-based, it is subject to the self-invocation caveat.

## Interview Notes

* Explain the Spring Security filter chain and `SecurityContextHolder`.
* Describe how `@EnableMethodSecurity` works and why proxies are needed.
* Know the difference between JWT resource servers and OAuth2 client login.
