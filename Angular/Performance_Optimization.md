# Angular Performance Optimization

## A Mental Model for Performance

Angular performance splits into **load performance** (how fast the app starts) and **runtime performance** (how smoothly it updates). Optimize load with lazy loading, bundle reduction, SSR, and image handling; optimize runtime with `OnPush`/signals, efficient lists, and avoiding wasted change detection. Always **measure first** (Lighthouse, DevTools Performance, `ng build --stats-json` + a bundle analyzer) before optimizing.

## Lazy Loading and Code Splitting

Load only what's needed for the current view. Route-level lazy loading (`loadComponent`/`loadChildren`) splits features into separate chunks fetched on navigation, shrinking the initial bundle.

```ts
{ path: 'reports', loadComponent: () => import('./reports.component').then(m => m.ReportsComponent) },
```

Combine with a **preloading strategy** to fetch likely-next chunks in the background after startup.

## Deferrable Views (`@defer`)

`@defer` lazily loads a template block's dependencies and renders on a trigger — deferring heavy, below-the-fold, or rarely-used UI out of the initial bundle.

```html
@defer (on viewport; prefetch on idle) {
  <heavy-dashboard />
} @placeholder { <skeleton /> } @loading { <spinner /> }
```

Triggers: `on idle`, `on viewport`, `on interaction`, `on hover`, `on timer`, and `when <expr>`. This is one of the most effective modern load-time optimizations.

## `OnPush` and Signals

Use `ChangeDetectionStrategy.OnPush` (or zoneless + signals) so components re-render only when their inputs/signals actually change, pruning most of the CD tree. Signals give fine-grained updates — only views reading a changed signal update. This is the single biggest runtime CD optimization. It requires **immutable data** so reference checks work.

## Efficient Lists: `track`

In `@for`, always provide a good `track` expression (a stable unique id). It lets Angular reuse DOM nodes instead of destroying/recreating them on every change — critical for large or frequently-updated lists (the successor to `trackBy`).

```html
@for (row of rows(); track row.id) { <app-row [row]="row" /> }
```

Tracking by index or a non-stable value defeats the optimization.

## Images: `NgOptimizedImage`

The `NgOptimizedImage` directive (`ngSrc`) optimizes image loading: automatic lazy loading, `srcset` generation, priority hints for LCP images, and warnings for oversized images — improving Largest Contentful Paint.

```html
<img ngSrc="hero.jpg" width="1200" height="600" priority />   <!-- priority = eager LCP -->
```

## Bundle Size and Tree Shaking

* Keep dependencies lean; prefer tree-shakable, standalone APIs (`provideX`) over large modules.
* Set **budgets** in `angular.json` to fail the build when bundles grow beyond thresholds.
* Analyze bundles (`source-map-explorer`, esbuild metafile) to find bloat.
* Standalone components and `providedIn: 'root'` services are tree-shakable — unused code is dropped.

## Virtual Scrolling and Pagination

Rendering thousands of DOM nodes is slow. The CDK `<cdk-virtual-scroll-viewport>` renders only the visible items, recycling nodes as you scroll. For big data sets, prefer virtual scrolling, pagination, or windowing over rendering everything.

## Avoiding Wasted Work in Templates

* Don't call functions/getters in templates that do heavy work — they run every CD cycle. Use `computed` signals or pure pipes to memoize.
* Prefer `async` pipe / `toSignal` over manual subscriptions.
* Move high-frequency non-UI work outside Angular with `runOutsideAngular`.
* Debounce expensive reactions (search, resize) with RxJS operators.

## SSR, Hydration, and Rendering

Server-Side Rendering with **hydration** improves perceived load and Core Web Vitals by shipping server-rendered HTML that the client reuses (rather than re-rendering). **Incremental hydration** (`@defer (hydrate on ...)`) hydrates parts of the page on demand, reducing initial JS execution (see SSR & Hydration).

## Interview Q&A

**Q: What are the main levers for Angular load performance?**
A: Route-level lazy loading, `@defer` deferrable views, bundle reduction/tree shaking with budgets, `NgOptimizedImage`, and SSR with hydration.

**Q: What's the biggest runtime change-detection optimization?**
A: `OnPush` (or zoneless + signals) so components update only when inputs/signals change, plus fine-grained signal reactivity — pruning most of the CD tree. It requires immutable data.

**Q: Why is `track` in `@for` important for performance?**
A: It identifies items by a stable key so Angular reuses existing DOM nodes rather than destroying and recreating them on each change — essential for large/updating lists.

**Q: What does `@defer` do for performance?**
A: It lazily loads a block's dependencies and renders on a trigger (viewport, interaction, idle, etc.), keeping heavy/below-the-fold UI out of the initial bundle and render.

**Q: How does `NgOptimizedImage` help?**
A: It adds lazy loading, `srcset`/responsive sizing, LCP priority hints, and size warnings, improving image load and Largest Contentful Paint.

**Q: How do you keep bundles small?**
A: Lean dependencies, standalone tree-shakable APIs, lazy loading, bundle budgets in `angular.json`, and analyzing output with a bundle analyzer.

**Q: How do you render very large lists efficiently?**
A: Use CDK virtual scrolling (render only visible items) or pagination instead of rendering all nodes.

**Q: Why avoid function calls in templates?**
A: They execute on every change detection cycle; expensive ones cause jank. Use `computed` signals or pure pipes to memoize results.

**Q: How does SSR + hydration improve performance?**
A: The server sends rendered HTML the browser displays immediately; hydration reuses that DOM instead of re-rendering, improving FCP/LCP and interactivity, with incremental hydration reducing initial JS.

**Q: What should you do before optimizing?**
A: Measure — use Lighthouse, DevTools Performance, and bundle analysis to find real bottlenecks rather than guessing.

## Interview Notes

* Separate load vs runtime performance and always measure first.
* Lazy load routes + use `@defer`; add preloading and bundle budgets.
* Use `OnPush`/signals with immutable data; provide stable `@for` `track`.
* Use `NgOptimizedImage`, virtual scrolling, and avoid heavy template function calls.
* Prefer `async`/`toSignal`; move hot work out with `runOutsideAngular`.
* Leverage SSR + (incremental) hydration for Core Web Vitals.
