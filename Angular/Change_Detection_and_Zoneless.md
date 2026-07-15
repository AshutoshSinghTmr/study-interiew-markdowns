# Angular Change Detection and Zoneless

## What Change Detection Does

Change detection (CD) is the process that keeps the DOM in sync with component state. After some event changes data, Angular walks the component tree, re-evaluates template bindings, and updates the DOM where values changed. Understanding **when** CD runs — and how to limit it — is central to Angular performance.

## The Role of zone.js

Traditionally Angular uses **zone.js**, which monkey-patches async browser APIs (`setTimeout`, `addEventListener`, `Promise`, XHR/fetch). When any patched async task completes, zone.js notifies Angular to run change detection across the tree. This is why bindings "just update" after clicks, timers, or HTTP responses without manual work — but it also means CD can run more often than necessary.

```ts
// zone.js triggers CD after this click handler runs, updating the view automatically
onClick() { this.count++; }
```

## Change Detection Strategies

Each component has a CD strategy:

* **`Default`** (`CheckAlways`) — the component is checked on every CD cycle triggered anywhere in the app.
* **`OnPush`** (`CheckOnce`) — the component is checked only when: an `@Input` reference changes, an event fires in the component, an observable bound via `async` emits, or a signal it reads changes. Otherwise it's skipped.

```ts
@Component({ changeDetection: ChangeDetectionStrategy.OnPush, /* ... */ })
export class ProductCardComponent { product = input.required<Product>(); }
```

`OnPush` dramatically cuts work by pruning subtrees, but requires **immutable data** (replace references, don't mutate) so reference checks detect changes.

## `ChangeDetectorRef`

`ChangeDetectorRef` gives manual control for edge cases:

* `markForCheck()` — mark this component and ancestors to be checked in the next cycle (used after an async update in `OnPush`).
* `detectChanges()` — run CD synchronously on this component now.
* `detach()` / `reattach()` — remove/restore a component from the CD tree (for very hot components you update manually).

```ts
private cdr = inject(ChangeDetectorRef);
this.service.data$.subscribe(d => { this.data = d; this.cdr.markForCheck(); });
```

With signals or the `async` pipe you rarely need `markForCheck` — they mark for check automatically.

## Signals and Change Detection

Signals bring **fine-grained** reactivity: when a signal read in a template changes, Angular knows exactly which views depend on it and can update just those — instead of dirty-checking the whole tree. This is the foundation for efficient `OnPush` and zoneless apps. Reading a signal in a template registers that template as a consumer, so no manual `markForCheck` is needed.

## Zoneless Change Detection

**Zoneless** Angular removes zone.js entirely. CD is instead triggered by explicit notifications — primarily **signal** changes, plus `async` pipe emissions, event bindings, and `markForCheck`. Enable it with `provideZonelessChangeDetection()` and remove `zone.js` from polyfills.

```ts
export const appConfig: ApplicationConfig = {
  providers: [provideZonelessChangeDetection()],
};
```

Benefits: smaller bundle (no zone.js), better performance and interoperability (no monkey-patching), clearer, more predictable CD. It works best when your app drives updates through signals; code relying on implicit zone-triggered CD (mutating fields after `setTimeout` without a signal) must be updated.

## `NgZone.runOutsideAngular`

In zone-based apps, wrap high-frequency work that shouldn't trigger CD (animations, scroll/mousemove handlers, third-party libs) in `runOutsideAngular` to avoid flooding change detection, re-entering with `run()` only when you need a view update.

```ts
private zone = inject(NgZone);
this.zone.runOutsideAngular(() => {
  el.addEventListener('mousemove', () => { /* no CD per move */ });
});
```

## `ExpressionChangedAfterItHasBeenCheckedError`

This dev-mode error means a binding's value changed **after** Angular checked it in the same cycle — usually because a parent's value was mutated in a child lifecycle hook (`ngAfterViewInit`) or during rendering. Angular runs a second verification pass in dev mode to catch this. Fixes: move the update earlier (`ngOnInit`), defer it (`queueMicrotask`/`afterNextRender`), or restructure so values are stable when read. It signals a data-flow smell, not just a warning.

## Interview Q&A

**Q: What is change detection?**
A: The process of synchronizing the DOM with component state by re-evaluating template bindings and updating changed values across the component tree.

**Q: How does zone.js trigger change detection?**
A: It monkey-patches async APIs (timers, events, promises, XHR); when a patched task finishes, it notifies Angular to run CD — so views update automatically after async work.

**Q: `Default` vs `OnPush`?**
A: `Default` checks a component on every cycle; `OnPush` checks only on input reference change, a component event, an `async` emission, or a signal change — skipping unchanged subtrees for performance.

**Q: What does `OnPush` require to work correctly?**
A: Immutable data — replace object/array references instead of mutating — so Angular's reference comparison detects changes.

**Q: What do `markForCheck` and `detectChanges` do?**
A: `markForCheck` schedules this component and ancestors for the next cycle (used after async updates under `OnPush`); `detectChanges` runs CD synchronously now on this component.

**Q: How do signals change the CD model?**
A: They enable fine-grained reactivity — Angular updates only views that read a changed signal, and template signal reads auto-mark for check, removing manual `markForCheck` and dirty-checking the whole tree.

**Q: What is zoneless change detection and why use it?**
A: CD without zone.js, triggered by signals/`async`/events/`markForCheck` via `provideZonelessChangeDetection()`. It shrinks the bundle, improves performance and interop, and makes CD predictable.

**Q: When do you use `runOutsideAngular`?**
A: For high-frequency work (mousemove, scroll, animations, third-party libs) that shouldn't trigger CD; re-enter with `NgZone.run` only when a view update is needed.

**Q: What causes `ExpressionChangedAfterItHasBeenCheckedError`?**
A: A bound value changed after Angular checked it in the same cycle (often a parent value mutated in a child hook). Fix by updating earlier, deferring the change, or restructuring the data flow. It's a dev-mode-only check.

**Q: Do you need `markForCheck` with the `async` pipe?**
A: No — the `async` pipe marks the component for check on each emission automatically, which is why it pairs well with `OnPush`.

## Interview Notes

* Explain CD as DOM/state synchronization and zone.js as the traditional trigger.
* Contrast `Default` vs `OnPush` and the immutability requirement.
* Know `ChangeDetectorRef` methods and when each applies.
* Explain fine-grained signal reactivity and automatic marking.
* Describe zoneless CD (`provideZonelessChangeDetection`) and its benefits/requirements.
* Diagnose `ExpressionChangedAfterItHasBeenCheckedError` and use `runOutsideAngular` for hot paths.
