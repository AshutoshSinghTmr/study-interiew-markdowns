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

## Interview Notes

* Describe the request lifecycle through `DispatcherServlet` and `HandlerAdapter`.
* Explain how controller method arguments are resolved.
* Know how exception resolvers and `@ControllerAdvice` produce consistent API errors.
