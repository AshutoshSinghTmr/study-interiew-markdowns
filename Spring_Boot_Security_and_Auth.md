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

## Interview Notes

* Explain the Spring Security filter chain and `SecurityContextHolder`.
* Describe how `@EnableMethodSecurity` works and why proxies are needed.
* Know the difference between JWT resource servers and OAuth2 client login.
