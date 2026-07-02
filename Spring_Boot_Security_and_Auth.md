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

`@EnableMethodSecurity` or `@EnableGlobalMethodSecurity` registers security advisors that intercept method calls. `MethodSecurityInterceptor` evaluates expressions using `MethodSecurityExpressionHandler`.

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
* custom `SecurityFilterChain` beans instead of extending `WebSecurityConfigurerAdapter`
* `oauth2ResourceServer().jwt()` for resource servers
* `http.authorizeHttpRequests()` for URL rules

## Identity and session management

Spring Security manages sessions using `SessionManagementConfigurer` and can control concurrency with session fixation and maximum sessions.

## Advanced topics

### Custom authentication providers

Implement `AuthenticationProvider` to authenticate non-standard credentials or integrate external identity systems.

### Security context propagation

For async execution, use `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL` or reactive `SecurityContext` propagation in WebFlux.

### Audit and event handling

Spring Security publishes authentication events and can be extended with `AuthenticationEventPublisher` for auditing.

## Interview Notes

* Explain the Spring Security filter chain and `SecurityContextHolder`.
* Describe how `@EnableMethodSecurity` works and why proxies are needed.
* Know the difference between JWT resource servers and OAuth2 client login.
