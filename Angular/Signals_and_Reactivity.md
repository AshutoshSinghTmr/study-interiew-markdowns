# Angular Signals and Reactivity

## What Signals Are

A **signal** is a reactive value container that notifies consumers when it changes. Signals give Angular **fine-grained reactivity**: the framework tracks exactly which signals a computation or template reads, so only the affected parts update. Signals are synchronous, glitch-free, and pull-based (values are computed lazily on read), and they are the foundation of Angular's move toward zoneless change detection.

```ts
const count = signal(0);        // writable signal
count();                        // read -> 0
count.set(5);                   // set a new value
count.update(n => n + 1);       // derive from current -> 6
```

Reading a signal in a template or reactive context **registers a dependency** automatically.

## `computed` — Derived State

`computed()` creates a read-only signal derived from other signals. It **memoizes**: it recomputes only when a dependency changes and only when read, and it caches the result.

```ts
const first = signal('Ada');
const last = signal('Lovelace');
const fullName = computed(() => `${first()} ${last()}`);   // recomputes when first/last change
```

Computeds form a dependency graph; because they're lazy and memoized, expensive derivations run only when actually needed.

## `effect` — Side Effects

`effect()` runs a side-effecting function whenever any signal it reads changes. Use effects for synchronization with non-reactive systems (logging, `localStorage`, third-party libs, manual DOM), **not** for deriving state (use `computed`).

```ts
constructor() {
  effect(() => localStorage.setItem('count', String(this.count())));
}
```

Effects run in an injection context and are cleaned up automatically with their component. Avoid setting signals inside effects (it's disallowed by default to prevent cycles; use `allowSignalWrites` only deliberately).

## Signal Inputs, Outputs, and Model

Signals integrate with component I/O (see Component Communication):

* `input()` / `input.required()` — read-only reactive inputs.
* `output()` — event emitter (not a signal, but the modern output API).
* `model()` — a writable signal supporting two-way binding `[(x)]`.

```ts
value = model(0);
double = computed(() => this.value() * 2);
```

## RxJS Interop: `toSignal` and `toObservable`

Signals and RxJS coexist. `@angular/core/rxjs-interop` bridges them:

* `toSignal(obs$)` — converts an Observable to a signal, subscribing and unsubscribing automatically (great for templates, replaces many `async` pipes).
* `toObservable(sig)` — converts a signal to an Observable for operator pipelines.

```ts
private query = signal('');
results = toSignal(
  toObservable(this.query).pipe(
    debounceTime(300),
    switchMap(q => this.api.search(q)),
  ),
  { initialValue: [] },
);
```

Use RxJS for async/event streams and time-based operators; use signals for synchronous state and templates.

## The `resource` API (Async Signals)

`resource()` / `rxResource()` model **asynchronous data as signals**: given a reactive request, they fetch and expose `value`, `status`, `error`, and support reloading — a signal-native alternative to manually wiring `switchMap` for data loading.

```ts
private id = signal(1);
user = resource({
  request: () => ({ id: this.id() }),
  loader: ({ request }) => fetch(`/api/users/${request.id}`).then(r => r.json()),
});
// template: @if (user.value(); as u) { {{ u.name }} }
```

## `linkedSignal` — Writable Derived State

`linkedSignal()` creates a writable signal whose value is derived from a source but can also be locally overridden, resetting when the source changes — useful for selection state that depends on a list.

```ts
selected = linkedSignal(() => this.options()[0]);   // resets when options change, but is settable
```

## Signals vs RxJS: When to Use Which

* **Signals** — synchronous component/UI state, derived values, template bindings, and driving change detection. Simpler mental model, no subscriptions.
* **RxJS** — asynchronous streams, events over time, HTTP, and complex coordination (debounce, retry, combine, cancellation).

They're complementary; convert at the boundary with `toSignal`/`toObservable`. Modern Angular guidance is "signals for state, RxJS for streams."

## Interview Q&A

**Q: What is a signal and what problem does it solve?**
A: A reactive value container that tracks its readers, enabling fine-grained updates — only computations/views that read a changed signal re-run. It underpins efficient, zoneless-friendly change detection.

**Q: `computed` vs `effect`?**
A: `computed` produces derived, memoized, read-only state and should be pure. `effect` performs side effects when its dependencies change and returns nothing. Never derive state in an effect.

**Q: Are signals synchronous or asynchronous?**
A: Synchronous and pull-based — reading returns the current value immediately, and `computed` values are lazily recomputed on read after a dependency changes.

**Q: How do signals integrate with component inputs?**
A: Via `input()`/`input.required()` (read-only reactive inputs) and `model()` (writable, two-way-bindable) — replacing `@Input`/`@Output` decorators in modern code.

**Q: How do you bridge signals and RxJS?**
A: `toSignal(observable$)` subscribes and exposes a signal (auto-unsubscribing); `toObservable(signal)` emits an observable of the signal's values for operator pipelines.

**Q: When should you use RxJS instead of signals?**
A: For asynchronous event streams and time-based/coordination operators (debounce, switchMap, retry, combineLatest, cancellation). Use signals for synchronous state and templates.

**Q: What is the `resource` API?**
A: A signal-native way to load async data: given a reactive request, it fetches via a loader and exposes `value`/`status`/`error` with reload support, avoiding manual `switchMap` wiring.

**Q: Why can't you (by default) set a signal inside an effect?**
A: To prevent infinite change cycles and make data flow predictable; deriving state should use `computed`. You can opt in with `allowSignalWrites` when truly needed.

**Q: What does `computed` memoization give you?**
A: It recomputes only when dependencies change and only when read, caching the result — so expensive derivations don't run on every change detection.

**Q: What is `linkedSignal`?**
A: A writable signal derived from a source that resets when the source changes but can be locally overridden — handy for dependent, user-editable selection state.

## Interview Notes

* Define signals and fine-grained, pull-based, glitch-free reactivity.
* Distinguish `signal`/`computed`/`effect` and the "no state derivation in effects" rule.
* Know signal I/O: `input()`, `output()`, `model()`.
* Explain `toSignal`/`toObservable` interop and the "signals for state, RxJS for streams" guidance.
* Mention `resource()` for async data and `linkedSignal` for writable derived state.
