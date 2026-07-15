# Angular Components and Templates

## Anatomy of a Component

A component is a TypeScript class decorated with `@Component`, associating a **selector**, a **template**, and optional **styles**. It is the unit of UI and encapsulates its view, logic, and (via view encapsulation) its CSS.

```ts
@Component({
  selector: 'app-counter',
  template: `<button (click)="inc()">{{ count() }}</button>`,
  styles: `button { font-weight: bold; }`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class CounterComponent {
  count = signal(0);
  inc() { this.count.update(n => n + 1); }
}
```

Prefer `OnPush` (or zoneless + signals) for performance. Templates can be inline (`template`) or external (`templateUrl`).

## Data Binding

Angular has four binding forms tying the class and template together:

* **Interpolation** `{{ expr }}` — render a value as text.
* **Property binding** `[prop]="expr"` — set a DOM/component property.
* **Event binding** `(event)="handler($event)"` — respond to events.
* **Two-way binding** `[(ngModel)]="value"` / `[(model)]="value"` — combine property + event (`[x]` + `(xChange)`).

```html
<input [value]="name()" (input)="name.set($any($event.target).value)" />
<app-child [(size)]="size" />   <!-- two-way on a model() input -->
```

Bindings are re-evaluated during change detection; keep template expressions cheap and side-effect free.

## The New Control Flow (Angular 17+)

Built-in block control flow replaces the older `*ngIf`/`*ngFor`/`*ngSwitch` structural directives. It's faster, type-checked, and needs no imports.

```html
@if (user(); as u) {
  <p>Welcome {{ u.name }}</p>
} @else {
  <p>Please log in</p>
}

@for (item of items(); track item.id) {   <!-- track is required -->
  <li>{{ item.label }}</li>
} @empty {
  <li>No items</li>
}

@switch (status()) {
  @case ('loading') { <spinner /> }
  @case ('error')   { <error-msg /> }
  @default          { <content /> }
}
```

`track` is mandatory in `@for` and is critical for performance — it identifies items so Angular reuses DOM nodes instead of recreating them (the successor to `trackBy`).

## Deferrable Views (`@defer`)

`@defer` lazily loads a block's dependencies and renders it on a trigger, improving initial load. Blocks: `@placeholder`, `@loading`, `@error`.

```html
@defer (on viewport) {
  <heavy-chart [data]="data()" />
} @placeholder {
  <div>Scroll to load…</div>
} @loading (after 100ms) {
  <spinner />
}
```

Triggers include `on idle` (default), `on viewport`, `on interaction`, `on hover`, `on timer(...)`, and `when <condition>`; `prefetch` can warm dependencies ahead of display.

## Lifecycle Hooks

Angular calls lifecycle hooks at defined moments:

* `ngOnChanges(changes)` — when a data-bound `@Input` changes (before `ngOnInit`).
* `ngOnInit()` — once after the first `ngOnChanges`; do initialization here, not in the constructor.
* `ngDoCheck()` — every change detection run (use sparingly).
* `ngAfterContentInit` / `ngAfterContentChecked` — projected content initialized/checked.
* `ngAfterViewInit` / `ngAfterViewChecked` — the component's view (and child views) initialized/checked; `@ViewChild` refs are available here.
* `ngOnDestroy()` — cleanup (unsubscribe, clear timers) before the component is destroyed.

Newer render hooks `afterNextRender` and `afterRender` (run only in the browser) are the modern place for DOM measurement/manipulation, safe for SSR.

```ts
constructor() {
  afterNextRender(() => this.chart = new Chart(this.el.nativeElement));
}
```

Prefer the constructor for DI, `ngOnInit` for setup, and `ngOnDestroy` (or `takeUntilDestroyed`/`DestroyRef`) for teardown.

## View Encapsulation

Component styles are scoped by **`ViewEncapsulation`**:

* **`Emulated`** (default) — Angular adds attribute selectors so styles apply only to this component (no real Shadow DOM).
* **`ShadowDom`** — uses native Shadow DOM for true isolation.
* **`None`** — styles are global (leak to the whole app).

## Template Reference Variables and `@let`

`#ref` captures a DOM element or component instance for use elsewhere in the template. `@let` (newer) declares a local template variable for a computed value.

```html
<input #box (keyup)="0" />
<p>{{ box.value }}</p>

@let total = price() * qty();
<p>Total: {{ total }}</p>
```

## Interview Q&A

**Q: What are the four types of data binding?**
A: Interpolation `{{ }}`, property binding `[ ]`, event binding `( )`, and two-way binding `[( )]` (property + event combined).

**Q: What replaced `*ngIf`/`*ngFor` and why?**
A: The built-in control flow `@if`/`@for`/`@switch` (Angular 17+): faster, type-checked, no imports needed, and `@for` requires `track` for efficient DOM reuse.

**Q: Why is `track` in `@for` important?**
A: It uniquely identifies items so Angular reuses existing DOM nodes on changes instead of destroying and recreating them, greatly improving list performance (successor to `trackBy`).

**Q: What is `@defer` used for?**
A: Deferrable views lazily load and render a block on a trigger (`viewport`, `interaction`, `idle`, `timer`, `when`), reducing initial bundle/render cost, with `@placeholder`/`@loading`/`@error` states.

**Q: Explain the main lifecycle hooks and their order.**
A: `ngOnChanges` → `ngOnInit` → `ngDoCheck` → `ngAfterContentInit/Checked` → `ngAfterViewInit/Checked` → `ngOnDestroy`. Init in `ngOnInit`, cleanup in `ngOnDestroy`.

**Q: Constructor vs `ngOnInit`?**
A: Use the constructor for dependency injection and field setup; use `ngOnInit` for initialization that depends on inputs being set and for logic you want testable/predictable after the first change detection.

**Q: What are `afterNextRender`/`afterRender`?**
A: Render callbacks that run only in the browser (not during SSR), the modern, SSR-safe place to do direct DOM work like measuring or integrating third-party libraries.

**Q: What are the view encapsulation modes?**
A: `Emulated` (default, attribute-scoped styles), `ShadowDom` (native isolation), and `None` (global styles).

**Q: Where are `@ViewChild` references available?**
A: After the view initializes — in `ngAfterViewInit` (or via signal `viewChild()` queries which update reactively).

**Q: What is a template reference variable?**
A: A `#name` in the template capturing a DOM element or component instance for use elsewhere in that template.

## Interview Notes

* Describe the `@Component` structure and prefer `OnPush`/signals.
* Know all four binding types and that expressions must be cheap and side-effect free.
* Explain the new control flow and why `@for` `track` matters.
* Explain `@defer` triggers and states.
* Recite lifecycle hook order; init in `ngOnInit`, teardown in `ngOnDestroy`, DOM work in `afterNextRender`.
* Know the three view encapsulation modes.
