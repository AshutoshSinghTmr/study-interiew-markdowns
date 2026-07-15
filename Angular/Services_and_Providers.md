# Angular Services and Providers

## What a Service Is

A **service** is a plain, injectable TypeScript class that encapsulates non-UI concerns — data access, business logic, state, logging, caching — so components stay focused on presentation. Services are shared through dependency injection, making them the primary unit of reuse and the seam for testing.

```ts
@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  getUser(id: number) { return this.http.get<User>(`/api/users/${id}`); }
}
```

`@Injectable()` marks a class as available for DI and lets it inject its own dependencies.

## Singleton vs Scoped Services

Scope is decided by **where** the service is provided (see Dependency Injection):

* `providedIn: 'root'` → one app-wide singleton (default, tree-shakable). Ideal for shared state and stateless helpers.
* A component's `providers` → a fresh instance per component instance, torn down with the component. Ideal for per-widget/per-feature state that shouldn't leak globally.
* A route's `providers` → scoped to a lazily-loaded feature.

Understanding scope prevents accidental shared state (or accidental duplication) — a very common bug source.

## Stateful Services (Modern Patterns)

A service is the standard place to hold state shared across components. Two idioms:

**Signal-based (modern, preferred):** expose readable signals and mutate through methods.

```ts
@Injectable({ providedIn: 'root' })
export class CartService {
  private readonly _items = signal<Item[]>([]);
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price, 0));

  add(item: Item) { this._items.update(list => [...list, item]); }
  clear() { this._items.set([]); }
}
```

**RxJS-based (classic):** a private `BehaviorSubject` with a public `Observable`.

```ts
private items$ = new BehaviorSubject<Item[]>([]);
readonly items = this.items$.asObservable();
add(item: Item) { this.items$.next([...this.items$.value, item]); }
```

Keep the mutable subject/signal private and expose read-only access, so state changes only through the service's API.

## The Facade Pattern

A **facade** service wraps a more complex subsystem (a store, several services, HTTP calls) behind a simple, component-friendly API. It centralizes orchestration, keeps components thin, and makes the underlying implementation (e.g. swapping NgRx for signals) replaceable without touching components.

```ts
@Injectable({ providedIn: 'root' })
export class CheckoutFacade {
  private cart = inject(CartService);
  private orders = inject(OrderApi);
  readonly total = this.cart.total;
  placeOrder() { return this.orders.create(this.cart.items()); }
}
```

## Provider Functions (`provideX`)

Modern libraries expose **provider functions** instead of feature `NgModule`s. They return configured providers for `ApplicationConfig` or route/component `providers`, and are tree-shakable and composable.

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideAnimationsAsync(),
    provideZonelessChangeDetection(),
  ],
};
```

The `withX(...)` "feature" functions configure optional behaviour, keeping the base provider minimal and shaking out unused code.

## `APP_INITIALIZER` / Bootstrapping Work

To run async setup before the app renders (load config, feature flags), provide an initializer that returns a Promise/Observable. Modern Angular offers `provideAppInitializer(...)`.

```ts
provideAppInitializer(() => inject(ConfigService).load());
```

## Best Practices

* Default to `providedIn: 'root'`; scope to a component only when you need per-instance state.
* Keep services single-responsibility; split data access from state/orchestration.
* Expose read-only state (signals `asReadonly()` / observables `asObservable()`); mutate via methods.
* Prefer signals for new stateful services; use RxJS where streams/operators add value.
* Make services easy to test by injecting collaborators (no `new` inside).

## Interview Q&A

**Q: What is a service and why use one?**
A: An injectable class for non-UI concerns (data, state, logic) shared via DI. It keeps components focused on the view, promotes reuse, and provides a clean testing seam.

**Q: How is a service's scope/lifetime determined?**
A: By where it's provided: `providedIn: 'root'` is an app singleton; a component's `providers` gives one instance per component; a route's `providers` scopes it to that lazy feature.

**Q: How do you hold shared state in a service today?**
A: Expose read-only signals (`asReadonly()`) with `computed` derivations and mutate via methods; the classic alternative is a private `BehaviorSubject` exposed as an observable.

**Q: Why keep the subject/signal private?**
A: So state changes only through the service's controlled API, preserving invariants and preventing components from mutating shared state directly.

**Q: What is the facade pattern in Angular?**
A: A service that exposes a simple API over a complex subsystem (store, multiple services), keeping components thin and making the implementation swappable.

**Q: What are provider functions like `provideHttpClient`?**
A: Tree-shakable functions that return configured providers for standalone apps, replacing feature `NgModule`s; `withX()` features add optional behaviour.

**Q: How do you run async initialization before the app starts?**
A: Provide an app initializer (`provideAppInitializer`/`APP_INITIALIZER`) returning a Promise/Observable that Angular awaits before rendering.

**Q: When would you scope a service to a component instead of root?**
A: When each component instance needs isolated state (e.g. a data grid's selection/sort state) that must be created and destroyed with the component.

**Q: Signals vs `BehaviorSubject` for service state?**
A: Signals are simpler, synchronous, and integrate with change detection for new code; RxJS subjects are preferable when you need stream operators, async composition, or existing observable interop.

## Interview Notes

* Define services and `@Injectable`, and why they're the reuse/testing unit.
* Explain scope via provider location and the risks of getting it wrong.
* Show modern signal-based stateful services with read-only exposure.
* Describe the facade pattern and provider functions (`provideX` + `withX`).
* Mention app initializers for pre-render async setup.
* State best practices: `providedIn: 'root'` default, single responsibility, inject collaborators.
