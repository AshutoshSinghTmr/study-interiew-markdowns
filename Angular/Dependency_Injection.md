# Angular Dependency Injection

## Why Dependency Injection

Dependency Injection (DI) is one of Angular's core features: instead of a class creating its collaborators, it **declares** what it needs and the framework **provides** them. This decouples classes, centralizes configuration, and makes testing easy (swap real services for fakes). Angular's DI is hierarchical and integrated with the component tree.

## Injecting Dependencies

There are two ways to obtain a dependency:

* **Constructor injection** — parameters typed with the dependency's token.
* **The `inject()` function** — call `inject(Token)` in an injection context (field initializer, constructor, factory). This is the modern idiom and works well with inheritance and functional code (guards, interceptors).

```ts
export class OrderComponent {
  // classic
  constructor(private http: HttpClient) {}
}

export class OrderComponent2 {
  // modern
  private http = inject(HttpClient);
}
```

`inject()` must run in an injection context; outside one (e.g. in a callback later) it throws, unless you capture it earlier or use `runInInjectionContext`.

## Providers and What They Configure

A **provider** tells an injector how to create a dependency for a token. The provider recipes:

* `useClass` — instantiate a class (the default: `{ provide: Api, useClass: RealApi }`).
* `useValue` — supply a ready value/object (config, constants).
* `useFactory` — call a function (with `deps`) to build the value.
* `useExisting` — alias one token to another (share a single instance).

```ts
providers: [
  { provide: API_URL, useValue: 'https://api.example.com' },
  { provide: PaymentGateway, useClass: StripeGateway },
  { provide: Logger, useFactory: () => new Logger(inject(ConfigService).level) },
  { provide: OldService, useExisting: NewService },
];
```

## Injection Tokens

Interfaces don't exist at runtime, so non-class dependencies (config objects, primitives, functions) are provided under an **`InjectionToken`**.

```ts
export const API_URL = new InjectionToken<string>('API_URL');
// provide: { provide: API_URL, useValue: '/api' }
// inject:  private url = inject(API_URL);
```

Tokens can carry a tree-shakable default via a factory: `new InjectionToken('x', { providedIn: 'root', factory: () => 'default' })`.

## The Hierarchical Injector Tree

Angular has **two injector hierarchies** that together resolve dependencies:

* **Environment injectors** — the root (application) injector and route-level injectors, configured by `ApplicationConfig` providers, `provideX` functions, and route `providers`.
* **Element (node) injectors** — created per component/directive, configured by a component's `providers` array.

Resolution walks **up** from the requesting element injector through its parents, then into the environment injector chain, until it finds a provider or throws `NullInjectorError`.

## Scoping and `providedIn`

Where you register a provider determines its **scope and lifetime**:

* `@Injectable({ providedIn: 'root' })` — a single app-wide singleton, **tree-shakable** (removed if unused). This is the default and best choice for most services.
* Provided in a component's `providers` — a **new instance per component instance** (and its children), destroyed with the component. Useful for per-widget state.
* Provided in a route's `providers` — scoped to that lazily-loaded route subtree.

```ts
@Injectable({ providedIn: 'root' })     // app singleton
export class AuthService {}

@Component({ providers: [GridState] })   // one instance per grid component
export class GridComponent {}
```

## Resolution Modifiers

Decorators/flags refine how the injector resolves a token:

* `@Optional()` / `{ optional: true }` — return `null` instead of throwing if not found.
* `@Self()` — only look in the element's own injector.
* `@SkipSelf()` — skip the current injector and start at the parent.
* `@Host()` — stop searching at the host component's injector.

```ts
private parentForm = inject(ControlContainer, { optional: true, skipSelf: true });
```

These matter for directives that should read a parent's provider (e.g. form controls) or avoid self-resolution.

## Multi Providers

A `multi: true` provider contributes to an **array** of values under one token, letting many registrations coexist — how `HTTP_INTERCEPTORS`, router features, and validators are collected.

```ts
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
```

## Interview Q&A

**Q: What is dependency injection and why does Angular use it?**
A: A pattern where the framework supplies a class's dependencies rather than the class constructing them, giving loose coupling, centralized configuration, and easy testing via substitutable providers.

**Q: `inject()` vs constructor injection?**
A: Both retrieve dependencies. `inject()` is a function callable in an injection context (fields, factories, functional guards/interceptors) and works cleanly with inheritance; constructor injection lists dependencies as typed parameters.

**Q: What provider recipes exist?**
A: `useClass` (instantiate a class), `useValue` (supply a value), `useFactory` (build via a function with `deps`), and `useExisting` (alias to another token).

**Q: What is an `InjectionToken` and when do you need one?**
A: A runtime token used to provide/inject non-class dependencies (config, primitives, functions) since TypeScript interfaces don't exist at runtime.

**Q: Explain Angular's hierarchical injectors.**
A: There are environment injectors (root/route) and element injectors (per component/directive). Resolution walks up the element injectors, then the environment chain, until a provider is found or it throws.

**Q: What does `providedIn: 'root'` do?**
A: Registers a tree-shakable application-wide singleton — the default and preferred scope for most services.

**Q: How do you get a new service instance per component?**
A: List it in the component's `providers` array; each component instance (and its descendants) then gets its own instance, destroyed with the component.

**Q: What do `@Self`, `@SkipSelf`, `@Optional`, and `@Host` do?**
A: `@Self` restricts to the local injector; `@SkipSelf` starts at the parent; `@Optional` returns null instead of throwing; `@Host` stops at the host component's injector.

**Q: What is a multi provider?**
A: A provider with `multi: true` that contributes to an array under a token (e.g. `HTTP_INTERCEPTORS`), so multiple values coexist for one token.

**Q: What happens if no provider is found?**
A: Angular throws a `NullInjectorError` (unless `@Optional`), indicating the token wasn't provided anywhere up the injector chain.

## Interview Notes

* Explain DI's purpose (decoupling, testability) and the two ways to inject.
* Know the four provider recipes and when to use each.
* Explain `InjectionToken` for non-class dependencies.
* Describe the environment vs element injector hierarchy and bottom-up resolution.
* Contrast `providedIn: 'root'` singletons with component-scoped providers.
* Recall the resolution modifiers and multi providers (`HTTP_INTERCEPTORS`).
