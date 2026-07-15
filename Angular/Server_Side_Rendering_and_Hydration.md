# Angular Server-Side Rendering and Hydration

## Client-Side vs Server-Side Rendering

A default Angular app is **client-side rendered (CSR)**: the browser downloads a JS bundle, then Angular builds the DOM. This delays first paint and is poor for SEO (crawlers may see an empty shell). **Server-Side Rendering (SSR)** runs the app on the server to produce fully-formed HTML for the initial request, so users see content sooner and crawlers get real markup.

## Angular SSR (`@angular/ssr`)

Modern Angular SSR is provided by `@angular/ssr` (the successor to "Angular Universal"), enabled when creating an app (`ng new --ssr`) or added with `ng add @angular/ssr`. It sets up a Node/Express server that renders the app per request and serves the result, along with `provideClientHydration()` on the browser side.

```ts
// app.config.ts (browser)
export const appConfig: ApplicationConfig = {
  providers: [provideClientHydration(withEventReplay())],
};
```

## Why SSR

* **SEO** — crawlers receive complete HTML, improving indexing of content-heavy/public pages.
* **Performance / Core Web Vitals** — faster First Contentful Paint and Largest Contentful Paint because content appears before JS fully loads.
* **Social sharing** — link previews (Open Graph tags) work because meta tags are server-rendered.

The trade-off is server infrastructure/cost and added complexity (code must be server-safe).

## Hydration

Historically Angular **destroyed and re-rendered** the server HTML on the client, causing a flicker and wasted work. **Non-destructive hydration** (`provideClientHydration()`) instead **reuses** the server-rendered DOM: Angular walks the existing nodes, attaches event listeners and bindings, and takes over without rebuilding — eliminating flicker and improving performance. Hydration is the recommended default with SSR.

## Incremental Hydration

**Incremental hydration** (built on `@defer`) hydrates parts of the page **on demand** rather than all at once, reducing the JavaScript executed at startup. Blocks hydrate on triggers like `on viewport` or `on interaction`.

```html
@defer (hydrate on viewport) {
  <comments-section />        <!-- server-rendered, hydrated only when scrolled into view -->
}
```

This keeps the initial interactivity cost low while still delivering server-rendered HTML.

## Event Replay

With `withEventReplay()`, user interactions that happen **before** hydration completes are captured and replayed once the relevant part hydrates, so early clicks aren't lost — important because SSR HTML is interactive-looking before JS is ready.

## Transfer State

Without care, data fetched on the server is fetched **again** on the client after hydration. `TransferState` serializes server-fetched data into the HTML so the client reuses it instead of re-requesting. `HttpClient` integrates with this automatically (caching GET requests across the SSR→CSR boundary) via `withHttpTransferCacheOptions`.

## Writing Server-Safe Code

SSR runs in Node, where browser globals don't exist. Code that touches `window`, `document`, `localStorage`, or `navigator` directly will crash on the server. Guard such code:

* Use `isPlatformBrowser(inject(PLATFORM_ID))` to run browser-only logic conditionally.
* Do DOM work in `afterNextRender`/`afterRender` (browser-only render hooks) rather than in constructors or `ngOnInit`.
* Use `Renderer2` and Angular abstractions instead of direct DOM APIs.

```ts
private platformId = inject(PLATFORM_ID);
ngOnInit() { if (isPlatformBrowser(this.platformId)) { /* use window */ } }
```

## SSG / Prerendering

**Static Site Generation (SSG / prerendering)** renders routes to static HTML at **build time** instead of per request — ideal for content that rarely changes (marketing pages, docs, blogs). Configure prerendered routes in the build; it gives SSR's SEO/performance benefits with the simplicity and cheapness of static hosting. SSR (per-request) suits dynamic, personalized content; SSG suits static content.

## Common Pitfalls

* Accessing `window`/`document` without a platform guard (server crash).
* Long/blocking server work delaying Time To First Byte.
* Forgetting `TransferState`, causing duplicate data fetches.
* Timers/subscriptions not cleaned up on the server.
* Non-deterministic rendering causing hydration mismatches.

## Interview Q&A

**Q: What is SSR and why use it?**
A: Rendering the app on the server to send complete HTML for the initial request, improving SEO, First/Largest Contentful Paint, and social link previews — at the cost of server infrastructure and complexity.

**Q: What is hydration?**
A: The client reusing the server-rendered DOM — attaching listeners and bindings to existing nodes instead of destroying and re-rendering — removing flicker and wasted work. Enabled with `provideClientHydration()`.

**Q: What is non-destructive hydration vs the old behaviour?**
A: Old SSR re-rendered and replaced server HTML on the client (flicker/waste); non-destructive hydration walks and reuses the existing DOM, taking over seamlessly.

**Q: What is incremental hydration?**
A: Hydrating page sections on demand via `@defer (hydrate on ...)` triggers (viewport, interaction), reducing startup JavaScript while keeping server-rendered content.

**Q: What is `TransferState` for?**
A: Serializing server-fetched data into the HTML so the client reuses it after hydration instead of refetching; `HttpClient` can cache GETs across the SSR→CSR boundary automatically.

**Q: How do you write server-safe code?**
A: Guard browser-only APIs with `isPlatformBrowser(PLATFORM_ID)`, do DOM work in `afterNextRender`/`afterRender`, and use `Renderer2` instead of direct `document`/`window` access.

**Q: What is event replay?**
A: Capturing user interactions that occur before hydration and replaying them once the relevant part hydrates (`withEventReplay()`), so early clicks aren't lost.

**Q: SSR vs SSG (prerendering)?**
A: SSR renders per request (dynamic/personalized content); SSG/prerendering renders routes to static HTML at build time (rarely-changing content) for cheap static hosting with the same SEO benefits.

**Q: What commonly breaks SSR?**
A: Accessing `window`/`document` without a platform guard, blocking server work, missing `TransferState` (double fetches), and non-deterministic output causing hydration mismatches.

## Interview Notes

* Contrast CSR vs SSR and state SSR's SEO/performance benefits and costs.
* Know `@angular/ssr` + `provideClientHydration()` and non-destructive hydration.
* Explain incremental hydration (`@defer hydrate on`) and event replay.
* Use `TransferState` to avoid duplicate fetches.
* Write server-safe code with `isPlatformBrowser` and `afterNextRender`.
* Distinguish SSR (per-request) from SSG/prerendering (build-time).
