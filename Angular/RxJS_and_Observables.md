# Angular RxJS and Observables

## Why RxJS in Angular

RxJS is Angular's library for **reactive, asynchronous streams**. The `HttpClient`, router events, reactive forms `valueChanges`, and many APIs return **Observables**. RxJS excels at events over time, cancellation, and composing async work with operators — complementing signals, which handle synchronous state.

## Observable vs Promise

* A **Promise** resolves once with a single value and is eager (starts immediately).
* An **Observable** can emit **many** values over time, is **lazy** (nothing happens until you subscribe), and is **cancellable** (unsubscribe stops work). It also supports rich operators.

```ts
const obs$ = new Observable<number>(sub => {
  sub.next(1); sub.next(2); sub.complete();
});
const sub = obs$.subscribe({ next: v => console.log(v), error: e => {}, complete: () => {} });
```

## Cold vs Hot Observables

* **Cold** — produces its data per subscriber; each subscription re-runs the producer (e.g. `HttpClient` requests — subscribing twice makes two calls).
* **Hot** — shares one producer across subscribers (e.g. DOM events, a `Subject`). Multicasting operators (`share`, `shareReplay`) turn a cold observable hot.

This distinction explains duplicate HTTP calls and is a frequent interview point.

## Subjects

A **Subject** is both an Observable and an Observer — you can `next()` values into it and subscribe to it. Variants:

* **`Subject`** — no initial value; only emits to current subscribers.
* **`BehaviorSubject`** — holds a current value, emits it immediately to new subscribers (ideal for state).
* **`ReplaySubject`** — replays the last N (or all) values to new subscribers.
* **`AsyncSubject`** — emits only the final value on completion.

```ts
private state$ = new BehaviorSubject<User | null>(null);
readonly user$ = this.state$.asObservable();
setUser(u: User) { this.state$.next(u); }
```

## Core Operators

Operators transform streams in a `pipe(...)`:

* **Transformation** — `map`, `scan`, `pluck`.
* **Filtering** — `filter`, `take`, `takeUntil`, `distinctUntilChanged`, `debounceTime`.
* **Combination** — `combineLatest`, `forkJoin`, `merge`, `withLatestFrom`, `startWith`.
* **Flattening (higher-order)** — `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`.
* **Error handling** — `catchError`, `retry`, `retryWhen`.

```ts
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.api.search(term)),
  catchError(() => of([])),
).subscribe(results => this.results.set(results));
```

## Choosing a Flattening Operator

The higher-order mapping operators differ in how they handle overlapping inner observables:

* **`switchMap`** — cancels the previous inner observable when a new value arrives. Best for search/type-ahead and navigation (only the latest matters).
* **`mergeMap`** — runs all inner observables concurrently. Good for independent parallel work.
* **`concatMap`** — queues inner observables, running one at a time in order. Best when order matters (sequential writes).
* **`exhaustMap`** — ignores new values while an inner observable is active. Best for preventing duplicate submits (e.g. login button spam).

Picking the wrong one causes race conditions or dropped/duplicated requests — a classic bug and interview question.

## The `async` Pipe

The `async` pipe subscribes in the template, renders the latest value, marks the component for check, and **auto-unsubscribes** on destroy — the safest way to consume observables in views and a strong pairing with `OnPush`.

```html
@if (user$ | async; as user) { <p>{{ user.name }}</p> }
```

## Subscription Management and Leaks

Manual subscriptions must be cleaned up or they leak (and keep running after the component is destroyed). Options, best first:

* Use the **`async` pipe** or **`toSignal`** so Angular manages the subscription.
* Use **`takeUntilDestroyed()`** (from `rxjs-interop`) to auto-complete on destroy.
* Use `takeUntil(this.destroy$)` with a `Subject` in `ngOnDestroy` (classic).

```ts
this.source$.pipe(takeUntilDestroyed()).subscribe(v => this.handle(v));
```

Avoid nested `subscribe` calls — flatten with `switchMap`/`concatMap` instead.

## Error Handling

`catchError` intercepts errors and can return a fallback observable (`of(...)`) or rethrow. `retry(n)` resubscribes on error. Because an error **terminates** the stream, handle it inside the inner observable (e.g. within `switchMap`) if you want the outer stream to survive subsequent events.

## Interview Q&A

**Q: Observable vs Promise?**
A: A Promise is eager and resolves once; an Observable is lazy, can emit many values over time, is cancellable via unsubscribe, and supports operators for composition.

**Q: What are cold and hot observables?**
A: Cold observables run their producer per subscriber (e.g. HTTP calls re-execute); hot observables share one producer among subscribers (e.g. events). `share`/`shareReplay` multicast a cold source.

**Q: What is a Subject, and how do the variants differ?**
A: A Subject is both observable and observer. `Subject` has no initial value; `BehaviorSubject` stores and emits a current value; `ReplaySubject` replays past values; `AsyncSubject` emits only the last value on completion.

**Q: Explain `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`.**
A: `switchMap` cancels the previous inner stream (latest wins — search); `mergeMap` runs all concurrently (parallel); `concatMap` queues sequentially (order matters); `exhaustMap` ignores new emissions while one is active (prevent double submit).

**Q: How do you avoid memory leaks from subscriptions?**
A: Prefer the `async` pipe or `toSignal` (Angular manages it); otherwise use `takeUntilDestroyed()` or `takeUntil(destroy$)` in `ngOnDestroy`.

**Q: Why might an HTTP call fire twice?**
A: `HttpClient` observables are cold — each subscription triggers a new request. Subscribing twice (or `async` pipe used twice) causes duplicate calls; use `shareReplay` to multicast.

**Q: What does the `async` pipe do for you?**
A: Subscribes, renders the latest value, triggers change detection, and unsubscribes automatically on destroy — removing manual subscription management.

**Q: How does error handling work in a stream?**
A: `catchError` intercepts and can return a fallback or rethrow; an error terminates the stream, so handle it in the inner observable (inside `switchMap`) to keep the outer stream alive.

**Q: How do signals and RxJS relate?**
A: Complementary — RxJS for async streams and time-based operators, signals for synchronous state and templates; bridge with `toSignal`/`toObservable`.

**Q: Why avoid nested subscribes?**
A: They leak, lose cancellation, and are hard to read; flatten with higher-order operators (`switchMap`/`concatMap`) instead.

## Interview Notes

* Contrast Observable vs Promise (lazy, multi-value, cancellable) and cold vs hot.
* Know the Subject variants and when `BehaviorSubject` fits state.
* Be fluent on the four flattening operators and their use cases.
* Explain subscription cleanup: `async` pipe/`toSignal` → `takeUntilDestroyed` → `takeUntil`.
* Explain `catchError`/`retry` and that errors terminate a stream.
* Position RxJS (streams) alongside signals (state) with interop.
