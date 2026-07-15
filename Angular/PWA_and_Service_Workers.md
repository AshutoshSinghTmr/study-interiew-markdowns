# Angular PWA and Service Workers

## What a PWA Is

A **Progressive Web App (PWA)** is a web app that uses modern browser capabilities — a **service worker**, a **web app manifest**, and HTTPS — to be installable, work offline, and behave more like a native app (home-screen icon, splash screen, push notifications). Angular provides `@angular/service-worker` and the `@angular/pwa` schematic to add these with minimal setup.

## Adding PWA Support

```bash
ng add @angular/pwa
```

This schematic:

* adds `@angular/service-worker` and registers it in the app config;
* creates `ngsw-config.json` (the service worker configuration);
* adds a `manifest.webmanifest` and icons;
* wires up `<link rel="manifest">` and theme color.

```ts
provideServiceWorker('ngsw-worker.js', {
  enabled: !isDevMode(),
  registrationStrategy: 'registerWhenStable:30000',
});
```

Service workers run only in production builds over HTTPS (or `localhost`).

## The Angular Service Worker (`ngsw`)

The service worker is a script the browser runs **in the background**, separate from the page, acting as a programmable network proxy. Angular's service worker is **configuration-driven** via `ngsw-config.json` (you don't hand-write the worker). It caches the app shell and assets so the app loads instantly and works offline, and it manages updates safely.

## Caching Strategies

`ngsw-config.json` has two main sections:

* **`assetGroups`** — the app shell and static assets (JS/CSS/images/fonts). Modes:
  * `prefetch` — cache during install (available offline immediately) — used for the core app.
  * `lazy` — cache on first request — used for optional assets.
* **`dataGroups`** — API/data requests, with strategies:
  * `freshness` — network-first, falling back to cache when offline (dynamic data).
  * `performance` — cache-first with a max age (rarely-changing data), for speed.

```jsonc
"dataGroups": [{
  "name": "api",
  "urls": ["/api/**"],
  "cacheConfig": { "strategy": "freshness", "maxSize": 100, "maxAge": "1h", "timeout": "5s" }
}]
```

## Updates: `SwUpdate`

Because the app is cached, users could keep running an old version. The service worker downloads new versions in the background; `SwUpdate` lets you detect and apply updates gracefully — typically prompting the user to reload.

```ts
private updates = inject(SwUpdate);
constructor() {
  this.updates.versionUpdates
    .pipe(filter(e => e.type === 'VERSION_READY'))
    .subscribe(() => {
      if (confirm('New version available. Reload?')) document.location.reload();
    });
}
```

The service worker serves a **consistent version** to a loaded app (it won't mix old and new files), and switches to the new version on the next full load/activation — avoiding broken lazy-chunk loads.

## Push Notifications: `SwPush`

`SwPush` handles the Web Push API — subscribing users (with a VAPID public key), receiving push messages, and handling notification clicks — enabling native-like re-engagement.

```ts
const sub = await this.swPush.requestSubscription({ serverPublicKey: VAPID_PUBLIC_KEY });
this.swPush.messages.subscribe(msg => { /* handle push payload */ });
```

## The Web App Manifest

`manifest.webmanifest` declares install metadata: `name`/`short_name`, `icons` (multiple sizes), `start_url`, `display` (`standalone` for app-like chrome), `theme_color`, and `background_color`. This is what makes the app **installable** and controls the splash screen and home-screen appearance.

## Offline Support and Limitations

With prefetch asset groups, the app shell loads offline; data availability depends on your `dataGroups` strategy. Limitations to know:

* Service workers only work in production over HTTPS (or `localhost`) — not `ng serve`.
* Debugging requires the browser's Application/Service Worker tools; stale caches are a common gotcha.
* The service worker can't cache what it hasn't seen; first visit must be online.
* Not every app needs a PWA — add it when offline use, installability, or push notifications provide real value.

## When to Use a PWA

Good fits: field/mobile apps needing offline access, content apps benefiting from instant loads and installability, and apps wanting push re-engagement. If your app is always-online and desktop-first, the added caching/update complexity may not be worth it.

## Interview Q&A

**Q: What is a PWA?**
A: A web app using a service worker, web app manifest, and HTTPS to be installable, work offline, and offer native-like features (home-screen install, push notifications).

**Q: How do you add PWA support in Angular?**
A: Run `ng add @angular/pwa`, which adds the service worker, `ngsw-config.json`, a manifest and icons, and registers the worker (enabled in production).

**Q: What is the Angular service worker and how is it configured?**
A: A background script acting as a network proxy that caches the app and manages updates; it's configuration-driven via `ngsw-config.json` rather than hand-written.

**Q: What caching strategies does `ngsw` support?**
A: Asset groups with `prefetch` (cache at install) or `lazy` (cache on demand), and data groups with `freshness` (network-first) or `performance` (cache-first with max age).

**Q: How do you handle app updates in a PWA?**
A: Use `SwUpdate` to detect `VERSION_READY` and prompt the user to reload; the worker serves a consistent version and switches on the next activation to avoid mixing old/new files.

**Q: What does `SwPush` do?**
A: Manages Web Push — subscribing with a VAPID key, receiving push messages, and handling notification clicks for re-engagement.

**Q: What is the web app manifest?**
A: A JSON file (`manifest.webmanifest`) with install metadata (name, icons, `start_url`, `display`, theme/background colors) that makes the app installable and controls its appearance.

**Q: Why doesn't the service worker work with `ng serve`?**
A: It's enabled only in production builds over HTTPS (or `localhost`); dev mode disables it to avoid caching interfering with development.

**Q: What are common PWA pitfalls?**
A: Stale caches, first visit requiring network, debugging via Application tools, and adding PWA complexity where it isn't needed.

**Q: When should you build a PWA?**
A: When offline access, installability, instant repeat loads, or push notifications add real value — not for every always-online app.

## Interview Notes

* Define a PWA (service worker + manifest + HTTPS) and `ng add @angular/pwa`.
* Explain the configuration-driven `ngsw` worker and `ngsw-config.json`.
* Know asset groups (prefetch/lazy) and data groups (freshness/performance).
* Handle updates with `SwUpdate` (consistent versioning) and push with `SwPush`.
* Describe the manifest's role in installability.
* Note SW works only in prod/HTTPS and when a PWA is (and isn't) worth it.
