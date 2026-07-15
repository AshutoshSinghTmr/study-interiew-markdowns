# Angular Component Communication

## Parent → Child: Inputs

A parent passes data to a child through **inputs**. The classic form uses the `@Input()` decorator; modern Angular uses the **signal-based `input()`** function, which exposes the value as a read-only signal.

```ts
// Modern signal inputs
export class UserCardComponent {
  name = input.required<string>();       // required input
  role = input('guest');                 // with default
  fullName = computed(() => `${this.name()} (${this.role()})`);
}
```

```html
<app-user-card [name]="user().name" [role]="user().role" />
```

Signal inputs are read-only, reactive (usable in `computed`/`effect`), and support transforms (`input(0, { transform: numberAttribute })`) and aliases. The decorator form `@Input() name!: string;` still works and is common in existing code.

## Child → Parent: Outputs

A child notifies a parent through **outputs**. The classic form is `@Output() x = new EventEmitter<T>()`; the modern form is the `output()` function.

```ts
export class UserCardComponent {
  selected = output<User>();             // modern
  choose(u: User) { this.selected.emit(u); }
}
```

```html
<app-user-card (selected)="onSelected($event)" />
```

`output()` returns an emitter with `.emit()`; there's also `outputFromObservable(...)` to derive an output from a stream.

## Two-Way Binding with `model()`

`model()` creates a writable signal input that also emits a matching change event, enabling `[(x)]` banana-in-a-box syntax without a separate `@Output`.

```ts
export class SliderComponent {
  value = model(0);                      // two-way bindable
  bump() { this.value.update(v => v + 1); }   // updates parent too
}
```

```html
<app-slider [(value)]="volume" />
```

## View Queries: Accessing Child Elements/Components

To reach elements or child component instances in the template, use view queries:

* Classic decorators: `@ViewChild(ChildComponent)` / `@ViewChildren(...)`, available in `ngAfterViewInit`.
* Modern signal queries: `viewChild()` / `viewChildren()` — reactive and available without lifecycle timing gymnastics.

```ts
export class FormComponent {
  nameInput = viewChild<ElementRef<HTMLInputElement>>('name');   // signal query
  focus() { this.nameInput()?.nativeElement.focus(); }
}
```

Query by template reference (`'name'`), by component/directive type, or with `{ read: ... }` to select a specific token.

## Content Queries and Projection

**Content projection** lets a component render markup supplied by its parent via `<ng-content>` — the basis of reusable wrappers (cards, dialogs).

```ts
// card.component.ts template:
// <div class="card"><ng-content select="[header]"></ng-content>
//   <ng-content></ng-content></div>
```

```html
<app-card>
  <h2 header>Title</h2>          <!-- projected into the header slot -->
  <p>Body content</p>            <!-- projected into the default slot -->
</app-card>
```

Multi-slot projection uses `select="..."` selectors; `ngProjectAs` re-labels projected content. To access projected content in code, use **content queries**: `@ContentChild`/`@ContentChildren` or signal `contentChild()`/`contentChildren()`, available in `ngAfterContentInit`.

## Sharing State via Services

For communication between **unrelated** or deeply nested components, a shared **injectable service** is the idiomatic channel — holding a signal or `BehaviorSubject` that components read and update. This avoids brittle input/output chains ("prop drilling").

```ts
@Injectable({ providedIn: 'root' })
export class CartService {
  private items = signal<Item[]>([]);
  readonly count = computed(() => this.items().length);
  add(i: Item) { this.items.update(list => [...list, i]); }
}
```

Any component injecting `CartService` reads `count()` reactively and calls `add(...)` — a clean cross-tree communication pattern.

## Choosing a Communication Mechanism

* Parent ↔ direct child → inputs/outputs (or `model()` for two-way).
* Access a child's DOM/instance → view queries (`viewChild`).
* Wrap/compose external markup → content projection + content queries.
* Unrelated/distant components or shared app state → a service with signals/observables.

## Interview Q&A

**Q: How does a parent pass data to a child?**
A: Through inputs — `@Input()` (classic) or the signal-based `input()`/`input.required()` (modern, read-only reactive signal), bound with `[prop]="value"`.

**Q: How does a child communicate back to a parent?**
A: Through outputs — `@Output() EventEmitter` (classic) or `output<T>()` (modern), emitting events consumed with `(event)="handler($event)"`.

**Q: What does `model()` do?**
A: Creates a writable signal input that also emits a change event, enabling two-way binding `[(x)]` without a separate output.

**Q: `@ViewChild` vs `@ContentChild`?**
A: `@ViewChild` queries elements/components in the component's own template (ready in `ngAfterViewInit`); `@ContentChild` queries projected content passed by the parent (ready in `ngAfterContentInit`).

**Q: What are signal queries?**
A: `viewChild()`/`viewChildren()`/`contentChild()`/`contentChildren()` return signals for queried elements, updating reactively and avoiding lifecycle-timing issues of the decorator queries.

**Q: What is content projection?**
A: Rendering parent-supplied markup inside a component via `<ng-content>`, with multi-slot selection (`select="..."`) — the mechanism behind reusable container components.

**Q: How do distant/unrelated components communicate?**
A: Via a shared injectable service holding a signal or `BehaviorSubject`; components inject it to read and update shared state, avoiding long input/output chains.

**Q: How do you pass content into named slots?**
A: Use `<ng-content select="[attr]">` (or element/class selectors) in the component and mark projected nodes accordingly; `ngProjectAs` can relabel content for selection.

**Q: Are signal inputs writable?**
A: No — `input()` is read-only. Use `model()` when the child needs to write back a two-way-bound value.

## Interview Notes

* Contrast classic `@Input`/`@Output` with signal `input()`/`output()`/`model()`.
* Explain `model()` for two-way binding.
* Distinguish view queries (own template, `ngAfterViewInit`) from content queries (projected, `ngAfterContentInit`), plus their signal equivalents.
* Describe single- and multi-slot content projection with `ng-content`.
* Recommend a shared service (signals/`BehaviorSubject`) for cross-tree communication.
