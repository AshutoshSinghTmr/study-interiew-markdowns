# Spring Boot Controller and REST API

## Controller Basics

Spring Boot web applications typically use `@RestController`, which combines `@Controller` and `@ResponseBody` to simplify REST endpoints.

### Request mapping

* `@RequestMapping`
* `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`

`RequestMappingHandlerMapping` scans controller methods and builds a mapping registry of `HandlerMethod` objects keyed by patterns and HTTP methods.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orders;

    public OrderController(OrderService orders) {
        this.orders = orders;
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderDto> get(@PathVariable Long id) {
        return ResponseEntity.ok(orders.findById(id));
    }

    @PostMapping
    public ResponseEntity<OrderDto> create(@Valid @RequestBody CreateOrderRequest request) {
        OrderDto created = orders.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

## Request and Response Binding

Spring resolves controller arguments using `HandlerMethodArgumentResolver`s for:

* `@RequestParam`
* `@PathVariable`
* `@RequestBody`
* `@ModelAttribute`
* `Principal`
* `Pageable`

Response bodies are written by `HttpMessageConverter`s such as `MappingJackson2HttpMessageConverter`.

### Data binding and conversion

`WebDataBinder` and `ConversionService` convert request values into target types, support formatting, and validate input.

## Validation

Use `@Valid` or `@Validated` on controller parameters. Spring triggers JSR-380 validation via `MethodValidationInterceptor` and reports constraint violations through `BindingResult` or exception handlers.

## Exception Handling

### Controller advice

`@ControllerAdvice` (or `@RestControllerAdvice`) with `@ExceptionHandler` centralizes error handling. Boot 3 pairs well with RFC 7807 `ProblemDetail` responses.

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(EntityNotFoundException.class)
    public ProblemDetail handleNotFound(EntityNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setDetail("Validation failed");
        return pd;
    }
}
```

### Internal flow

1. Handler method throws an exception.
2. `HandlerExceptionResolver`s evaluate available handlers.
3. `ExceptionHandlerExceptionResolver` or `ResponseStatusExceptionResolver` converts the exception to a response.
4. `ResponseEntityExceptionHandler` provides defaults for common exceptions.

## Content Negotiation

`ContentNegotiationManager` chooses the response media type based on request headers, path extensions, or parameter values. Boot defaults to JSON using Jackson.

## Security in Controllers

Protect endpoints with URL-based security or method security annotations.

### Method security internals

`@EnableMethodSecurity` (Spring Security 6; it replaces the legacy `@EnableGlobalMethodSecurity`) registers security advisors and interceptors that evaluate expressions through `MethodSecurityExpressionHandler`. It enables `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, and `@PostFilter`.

## REST API Best Practices

* Keep controllers thin and delegate business logic to services.
* Use DTOs or API models instead of exposing entities.
* Version APIs using URL paths or headers.
* Use `ResponseEntity` for explicit status codes and headers.
* Prefer `@ControllerAdvice` for consistent error formats.

## HATEOAS and Hypermedia

Spring HATEOAS supports link-driven representations. `RepresentationModel`, `EntityModel`, and `CollectionModel` help build hypermedia responses for REST APIs.

## Filters and Interceptors

### Servlet filters

Servlet filters run before Spring MVC and can modify the request or response. Boot auto-registers filter beans and supports custom filter registration.

### Handler interceptors

`HandlerInterceptor` runs before and after controller execution and is registered via `WebMvcConfigurer`.

### Filter vs interceptor order

Filters execute before interceptors. Use filters for cross-cutting concerns such as security, CORS, and request logging.

## Internal Request Flow

1. The embedded server receives the HTTP request.
2. `DispatcherServlet` processes the request.
3. `HandlerMapping` locates a handler method.
4. `HandlerAdapter` invokes the method.
5. Argument resolvers bind parameters.
6. The controller returns a value.
7. A `HandlerMethodReturnValueHandler` processes the return value — `@ResponseBody`/`ResponseEntity` values are serialized by an `HttpMessageConverter`, while view names are resolved by a `ViewResolver` — and the response is committed.

## DispatcherServlet Initialization

On startup `DispatcherServlet.initStrategies` wires a set of **special beans** from the context (falling back to `DispatcherServlet.properties` defaults): `HandlerMapping`, `HandlerAdapter`, `HandlerExceptionResolver`, `ViewResolver`, `LocaleResolver`, `ThemeResolver`, `MultipartResolver`, and `FlashMapManager`. For annotated controllers, `RequestMappingHandlerMapping` builds a registry of `RequestMappingInfo -> HandlerMethod`, and `RequestMappingHandlerAdapter` invokes the method using its `HandlerMethodArgumentResolver`s and `HandlerMethodReturnValueHandler`s.

## Message Converters and Content Negotiation

Request/response bodies are handled by an ordered list of `HttpMessageConverter`s (Jackson for JSON, Jackson XML, `String`, byte array, form). The chosen converter depends on the request `Content-Type`/`Accept` and the `produces`/`consumes` of the mapping. `ContentNegotiationManager` decides the response media type — by `Accept` header by default (path-extension negotiation is off in Boot). Register custom converters or negotiation rules via `WebMvcConfigurer`.

## Validation Beyond `@Valid`

* **Groups**: `@Validated(OnCreate.class)` runs only constraints in that group, so create vs update can differ.
* **Cross-field**: a class-level custom `ConstraintValidator` validates relationships between fields.
* **Method validation**: `@Validated` on a bean validates method params/returns (`ConstraintViolationException`).
* **Errors**: `@RequestBody` failures raise `MethodArgumentNotValidException`; simple `@RequestParam`/`@PathVariable` failures raise `HandlerMethodValidationException` (Boot 3.2). Both should map to a 400.

## Error Handling with `ProblemDetail` (RFC 7807)

Boot 3 supports RFC 7807 responses (`spring.mvc.problemdetails.enabled=true`, or return a `ProblemDetail`). Extend `ResponseEntityExceptionHandler` to convert Spring MVC exceptions to `ProblemDetail`, and customize the fallback `/error` mapping via an `ErrorController`/`ErrorAttributes`. `ProblemDetail` carries `type`, `title`, `status`, `detail`, `instance`, plus custom properties.

## HTTP Caching and Conditional Requests

Return validators to avoid re-sending unchanged bodies: `ResponseEntity.ok().eTag(v).cacheControl(CacheControl.maxAge(Duration.ofMinutes(5))).body(...)`, or enable `ShallowEtagHeaderFilter`. Spring honors `If-None-Match`/`If-Modified-Since` and returns `304 Not Modified` when appropriate.

## Declarative and Fluent HTTP Clients

* **`RestClient`** (Boot 3.2) — synchronous fluent client that supersedes `RestTemplate` for new code.
* **`WebClient`** — non-blocking client (usable in MVC for parallel calls).
* **HTTP interfaces** — declare `@HttpExchange` methods and build a proxy with `HttpServiceProxyFactory` over `RestClient`/`WebClient`.

```java
interface CatalogClient {
    @GetExchange("/products/{id}")
    Product byId(@PathVariable String id);
}
```

## Async Request Processing

Return `Callable<T>`, `DeferredResult<T>`, or `WebAsyncTask<T>` to release the servlet thread while work happens elsewhere; stream with `ResponseBodyEmitter`, `SseEmitter` (server-sent events), or `StreamingResponseBody`. This scales long-running or push endpoints without tying up the container's request threads.

## Interview Q&A

**Q: What is the difference between `@Controller` and `@RestController`?**
A: `@RestController` is `@Controller` + `@ResponseBody`, so every handler's return value is serialized directly to the response body (typically JSON) instead of being resolved as a view name.

**Q: Walk through what `DispatcherServlet` does with a request.**
A: It finds the handler via `HandlerMapping`, invokes it through a `HandlerAdapter`, resolves arguments with `HandlerMethodArgumentResolver`s, then processes the return value with a `HandlerMethodReturnValueHandler` (message converter for bodies, view resolver for views), applying `HandlerExceptionResolver`s on error.

**Q: How does `@Valid` trigger validation and where do the errors go?**
A: A Bean Validation provider validates the annotated argument; failures raise `MethodArgumentNotValidException` (for `@RequestBody`) or bind to a `BindingResult`, which `@ExceptionHandler`/`ResponseEntityExceptionHandler` turn into a 400 response.

**Q: How do you return consistent error responses across an API?**
A: Centralize handling in a `@RestControllerAdvice` with `@ExceptionHandler` methods, and in Boot 3 return `ProblemDetail` (RFC 7807) for a standard error shape.

**Q: What is the difference between a servlet `Filter` and a `HandlerInterceptor`?**
A: Filters run in the servlet container before/after `DispatcherServlet` and see the raw request/response; interceptors run inside Spring MVC around the handler (`preHandle`/`postHandle`/`afterCompletion`) with access to the resolved `HandlerMethod`.

**Q: How should you version a REST API?**
A: Common options are URI versioning (`/api/v1/...`), a custom header, or content negotiation via the `Accept` media type; URI versioning is the simplest and most cache-friendly.

**Q: How does content negotiation pick JSON vs XML?**
A: `ContentNegotiationManager` inspects the request `Accept` header (path-extension and param strategies are off by default in Boot) and matches it against the mapping's `produces` and the available `HttpMessageConverter`s; JSON via Jackson is the default.

**Q: What is the difference between `@RequestParam`, `@PathVariable`, and `@RequestBody`?**
A: `@RequestParam` binds query/form parameters, `@PathVariable` binds URI template segments, and `@RequestBody` deserializes the request body (via a message converter) into an object.

**Q: How do you return a specific status code and headers cleanly?**
A: Return `ResponseEntity<T>` (e.g. `ResponseEntity.status(HttpStatus.CREATED).header(...).body(dto)`), or use `@ResponseStatus` on a method/exception for a fixed status.

**Q: How do you stream a large response or handle file uploads?**
A: Stream with `StreamingResponseBody`, `ResponseBodyEmitter`, or `SseEmitter`; accept uploads with `@RequestPart`/`MultipartFile` (configure `spring.servlet.multipart` limits).

**Q: `RestTemplate`, `WebClient`, or `RestClient` — which should you use?**
A: For new blocking code use `RestClient` (Boot 3.2, fluent, supersedes `RestTemplate`); use `WebClient` for reactive or high-concurrency parallel calls. `RestTemplate` is maintenance-only.

## Interview Notes

* Describe the request lifecycle through `DispatcherServlet` and `HandlerAdapter`.
* Explain how controller method arguments are resolved.
* Know how exception resolvers and `@ControllerAdvice` produce consistent API errors.
