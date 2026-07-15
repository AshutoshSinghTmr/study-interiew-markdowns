# Angular Directives and Pipes

## What Directives Are

A directive is a class that adds behaviour to elements. Components are technically directives with a template. There are two other kinds:

* **Attribute directives** change the appearance or behaviour of an existing element (e.g. `ngClass`, `ngStyle`, a custom highlight).
* **Structural directives** change the DOM layout by adding/removing elements (e.g. the legacy `*ngIf`/`*ngFor`, which the built-in `@if`/`@for` control flow now supersedes for the common cases).

## Built-in Attribute Directives

`ngClass` and `ngStyle` conditionally apply classes/styles; `ngModel` (from `FormsModule`) provides two-way binding for template-driven forms.

```html
<div [ngClass]="{ active: isActive(), disabled: isDisabled() }"></div>
<div [ngStyle]="{ color: color(), 'font-size.px': size() }"></div>
<input [(ngModel)]="name" />
```

Simple class/style toggles are often cleaner with direct bindings: `[class.active]="isActive()"` and `[style.color]="color()"`.

## Custom Attribute Directives

Create a directive with `@Directive`, using `HostBinding`/`HostListener` (or the `host` metadata) to bind to the host element's properties and events.

```ts
@Directive({ selector: '[appHighlight]' })
export class HighlightDirective {
  color = input('yellow', { alias: 'appHighlight' });
  @HostBinding('style.backgroundColor') bg = '';
  @HostListener('mouseenter') onEnter() { this.bg = this.color(); }
  @HostListener('mouseleave') onLeave() { this.bg = ''; }
}
```

```html
<p [appHighlight]="'lightblue'">Hover me</p>
```

Directives can inject the host `ElementRef`, `Renderer2` (for SSR-safe DOM changes), and other directives/components on the same element.

## Structural Directives and the Microsyntax

Structural directives use the `*` shorthand, which desugars to an `<ng-template>`. A custom structural directive receives a `TemplateRef` and a `ViewContainerRef` to stamp views in/out.

```ts
@Directive({ selector: '[appUnless]' })
export class UnlessDirective {
  private shown = false;
  constructor(private tpl: TemplateRef<unknown>, private vcr: ViewContainerRef) {}
  @Input() set appUnless(cond: boolean) {
    if (!cond && !this.shown) { this.vcr.createEmbeddedView(this.tpl); this.shown = true; }
    else if (cond) { this.vcr.clear(); this.shown = false; }
  }
}
```

For most conditionals/loops, prefer the built-in `@if`/`@for` blocks; write custom structural directives only for specialized rendering logic.

## Directive Composition and Host Directives

The **host directives** API lets a component/directive apply other directives to its host without wrapper elements — composing behaviour declaratively.

```ts
@Component({
  selector: 'app-menu',
  hostDirectives: [CdkMenu, { directive: HighlightDirective, inputs: ['appHighlight: color'] }],
  template: `…`,
})
export class MenuComponent {}
```

## What Pipes Are

A **pipe** transforms a value for display in a template using the `|` operator. Angular ships many: `date`, `currency`, `number`, `percent`, `json`, `slice`, `uppercase`/`lowercase`/`titlecase`, `keyvalue`, and the essential `async`.

```html
<p>{{ amount() | currency:'USD':'symbol':'1.2-2' }}</p>
<p>{{ createdAt() | date:'medium' }}</p>
<p>{{ user$ | async }}?.name</p>
```

## The `async` Pipe

`async` subscribes to an `Observable`/`Promise`, returns the latest value, and — crucially — **unsubscribes automatically** when the component is destroyed, preventing leaks and manual subscription management. It also marks the component for check on new values, which pairs well with `OnPush`.

## Custom Pipes: Pure vs Impure

A custom pipe implements `PipeTransform`. Pipes are **pure by default**: Angular re-runs `transform` only when the input reference changes — efficient and memoized-like. An **impure** pipe (`pure: false`) runs on every change detection cycle, which is costly and can hurt performance (the reason mutating arrays doesn't update a pure pipe — the reference is unchanged).

```ts
@Pipe({ name: 'truncate' })              // pure by default
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 20): string {
    return value.length > limit ? value.slice(0, limit) + '…' : value;
  }
}
```

Keep pipes pure and side-effect free; for expensive transforms prefer computing in the component (a `computed` signal) over an impure pipe.

## Directive vs Pipe vs Component

* **Component** — renders a template (UI).
* **Directive** — augments an existing element's behaviour/appearance without its own template.
* **Pipe** — transforms a value for display, invoked in template expressions.

## Interview Q&A

**Q: What are the kinds of directives?**
A: Components (directives with a template), attribute directives (change appearance/behaviour), and structural directives (change DOM layout by adding/removing elements).

**Q: How do you build a custom attribute directive?**
A: Use `@Directive` with a selector and bind to the host via `@HostBinding`/`@HostListener` (or `host` metadata), optionally injecting `ElementRef`/`Renderer2`.

**Q: How does a structural directive work internally?**
A: The `*` syntax desugars to an `<ng-template>`; the directive receives a `TemplateRef` and `ViewContainerRef` and stamps/clears embedded views based on its input.

**Q: What is a pipe and when should you use one?**
A: A declarative value transformer used in templates with `|`, for formatting/display logic like dates, currency, and async values.

**Q: Pure vs impure pipes?**
A: Pure pipes (default) re-run only when the input reference changes — efficient. Impure pipes (`pure: false`) run every change detection cycle — flexible but costly; use sparingly.

**Q: Why doesn't mutating an array update a pure pipe?**
A: Pure pipes detect change by reference; mutating in place keeps the same reference, so `transform` isn't re-invoked. Replace the reference (immutable update) instead.

**Q: What does the `async` pipe do?**
A: Subscribes to an observable/promise, renders the latest value, marks for check on updates, and auto-unsubscribes on destroy — eliminating manual subscription cleanup.

**Q: What are host directives?**
A: The directive composition API (`hostDirectives`) that applies other directives to a component/directive's host element, composing behaviour without wrapper elements.

**Q: When would you write a custom structural directive vs use `@if`/`@for`?**
A: Use the built-in control flow for ordinary conditionals/loops; write a custom structural directive only for specialized view-stamping logic not covered by the blocks.

## Interview Notes

* Classify components/attribute/structural directives and give examples.
* Build a custom attribute directive with `HostBinding`/`HostListener`.
* Explain structural directive desugaring to `ng-template` with `TemplateRef`/`ViewContainerRef`.
* Know common built-in pipes and the `async` pipe's auto-unsubscribe.
* Explain pure vs impure pipes and the reference-change detection nuance.
* Mention host directives for composition.
