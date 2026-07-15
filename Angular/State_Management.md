# Angular State Management

## Levels of State

Not all state needs a store. Match the tool to the scope:

* **Local component state** — a signal or field in the component. Simplest; use it whenever state doesn't need sharing.
* **Shared feature/app state** — a service holding signals or a `BehaviorSubject` (see Services). Covers most apps.
* **Complex global state** — a dedicated store (NgRx and friends) when you have many features, cross-cutting state, complex async flows, undo/redo, or need strict traceability.

Reaching for a heavy store too early is a common over-engineering mistake; start simple and escalate when pain appears.

## Signal-Based State Services (Modern Default)

For most applications, an injectable service exposing read-only signals with `computed` derivations is the cleanest state solution — no boilerplate, synchronous, and change-detection friendly.

```ts
@Injectable({ providedIn: 'root' })
export class TodoStore {
  private readonly _todos = signal<Todo[]>([]);
  readonly todos = this._todos.asReadonly();
  readonly remaining = computed(() => this._todos().filter(t => !t.done).length);

  add(title: string) { this._todos.update(t => [...t, { id: crypto.randomUUID(), title, done: false }]); }
  toggle(id: string) {
    this._todos.update(t => t.map(x => x.id === id ? { ...x, done: !x.done } : x));
  }
}
```

Note the **immutable updates** (spread/`map`) — replacing references is what makes change detection and `OnPush`/signals work correctly.

## NgRx: The Redux Pattern

**NgRx Store** implements the Redux pattern for Angular: a single immutable **state** tree, changed only by dispatching **actions**, processed by pure **reducers**, read via memoized **selectors**, with side effects (HTTP, etc.) isolated in **effects**.

```ts
// actions
export const loadUsers = createAction('[Users] Load');
export const loadUsersSuccess = createAction('[Users] Success', props<{ users: User[] }>());

// reducer (pure)
export const usersReducer = createReducer(initialState,
  on(loadUsers, s => ({ ...s, loading: true })),
  on(loadUsersSuccess, (s, { users }) => ({ ...s, loading: false, users })),
);

// selector (memoized)
export const selectUsers = createSelector(selectUsersState, s => s.users);

// effect (side effects)
loadUsers$ = createEffect(() => this.actions$.pipe(
  ofType(loadUsers),
  switchMap(() => this.api.getUsers().pipe(map(users => loadUsersSuccess({ users })))),
));
```

Benefits: predictability, time-travel debugging (Redux DevTools), testability of pure reducers, and clear separation of side effects. Cost: significant boilerplate and a learning curve.

## NgRx SignalStore (Modern NgRx)

**SignalStore** (`@ngrx/signals`) is the signal-based evolution of NgRx: a store built from signals with far less boilerplate, exposing state as signals and computed selectors, with methods and RxJS/promise-based updaters. It's the recommended NgRx style for new signal-first apps.

```ts
export const TodoStore = signalStore(
  { providedIn: 'root' },
  withState({ todos: [] as Todo[] }),
  withComputed(({ todos }) => ({ remaining: computed(() => todos().filter(t => !t.done).length) })),
  withMethods(store => ({
    add(title: string) { patchState(store, s => ({ todos: [...s.todos, newTodo(title)] })); },
  })),
);
```

## ComponentStore and Other Options

* **`@ngrx/component-store`** — a lightweight, local, per-component reactive store for complex component state without global boilerplate.
* **NGXS** — a more class/decorator-based state library, less boilerplate than classic NgRx.
* **Elf / Akita** — repository-style reactive stores.
* **Plain services + signals** — often all you need.

## Selectors, Immutability, and Devtools

Selectors are **memoized** pure functions that derive view models from state, recomputing only when inputs change — the read side of a store. State must be treated as **immutable**; reducers/updaters return new objects rather than mutating, which enables `OnPush`, predictable diffs, and time-travel. The **Redux DevTools** integration lets you inspect dispatched actions and step through state history.

## Choosing an Approach

* Small/medium app → signals in services (or SignalStore).
* Complex component-local state → ComponentStore.
* Large app, many teams, complex flows, auditability → NgRx Store/SignalStore.
* Always prefer the lightest tool that keeps state predictable and testable.

## Interview Q&A

**Q: When do you actually need a state management library?**
A: When state is shared across many features, involves complex async flows, needs auditability/time-travel, or component/service state becomes hard to reason about. Otherwise signals in services suffice.

**Q: What is the modern default for state in Angular?**
A: Injectable services exposing read-only signals with `computed` derivations and immutable updates — simple, synchronous, and change-detection friendly.

**Q: Explain the NgRx building blocks.**
A: Actions describe events, reducers are pure functions producing new state, selectors are memoized reads, and effects isolate side effects (HTTP) reacting to actions — over a single immutable store.

**Q: Why must reducers be pure and state immutable?**
A: Purity/immutability enable predictable updates, memoized selectors, `OnPush` change detection, easy testing, and time-travel debugging; mutation breaks all of these.

**Q: What is NgRx SignalStore?**
A: A signal-based store from `@ngrx/signals` with minimal boilerplate, exposing state and computed selectors as signals and mutating via `patchState` and methods — the modern NgRx style.

**Q: What is ComponentStore for?**
A: Managing complex local state for a single component reactively, without the ceremony or global scope of the full NgRx Store.

**Q: What do selectors give you?**
A: Memoized, composable, pure derivations of view models from state that recompute only when their inputs change, decoupling components from state shape.

**Q: What are the downsides of classic NgRx?**
A: Boilerplate and a learning curve; it can be overkill for small apps. SignalStore and plain signal services reduce that overhead.

**Q: How do you debug NgRx state?**
A: Via the Redux DevTools integration, inspecting dispatched actions and stepping through the state timeline (time-travel debugging).

## Interview Notes

* Match state scope to tooling: local signal → service signals → store.
* Show a signal-based state service with immutable updates as the modern default.
* Explain NgRx actions/reducers/selectors/effects and why purity/immutability matter.
* Know SignalStore and ComponentStore as lighter modern options.
* Emphasize memoized selectors and Redux DevTools.
* Warn against premature adoption of a heavy store.
