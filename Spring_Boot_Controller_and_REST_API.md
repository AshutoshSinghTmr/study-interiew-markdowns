# Spring Boot Controller and REST API

## Controller Basics

Spring Boot web applications typically use `@RestController`, which combines `@Controller` and `@ResponseBody` to simplify REST endpoints.

### Request mapping

* `@RequestMapping`
* `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`

`RequestMappingHandlerMapping` scans controller methods and builds a mapping registry of `HandlerMethod` objects keyed by patterns and HTTP methods.

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

`@ControllerAdvice` with `@ExceptionHandler` centralizes error handling.

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

`@EnableGlobalMethodSecurity` or `@EnableMethodSecurity` registers security advisors and interceptors that evaluate expressions through `MethodSecurityExpressionHandler`.

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
7. `HandlerResult` is transformed by `HttpMessageConverter` or `ViewResolver` and written to the response.

## Interview Notes

* Describe the request lifecycle through `DispatcherServlet` and `HandlerAdapter`.
* Explain how controller method arguments are resolved.
* Know how exception resolvers and `@ControllerAdvice` produce consistent API errors.
