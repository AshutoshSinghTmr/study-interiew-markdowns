# Angular Build and Deployment

## The Angular CLI Build System

The Angular CLI drives builds through **builders** configured in `angular.json`. Modern Angular uses the **application builder** (`@angular/build`) powered by **esbuild** for production bundling and **Vite** for the dev server — a major speedup over the legacy webpack-based builder. `ng build` produces optimized output in `dist/`; `ng serve` runs the fast dev server with hot module replacement.

```bash
ng serve                 # dev server (Vite), fast rebuilds
ng build                 # production build by default (AOT, optimized)
ng build --configuration development
```

## AOT Compilation

Production builds use **Ahead-of-Time (AOT)** compilation: templates and components are compiled to JavaScript at **build time** rather than in the browser. Benefits: faster startup (no runtime compiler shipped), smaller bundles, and **template errors caught at build time**. AOT is the default for `ng build`.

## Optimization: Tree Shaking, Minification, Bundling

Production builds automatically apply:

* **Tree shaking** — removing unused (dead) code; standalone APIs and `providedIn: 'root'` services are tree-shakable.
* **Minification & mangling** — shrinking JS/CSS.
* **Bundling & code splitting** — lazy routes and `@defer` blocks become separate chunks.
* **Content hashing** — filenames include hashes (`main.abc123.js`) for long-term caching / cache busting.

## Build Configurations and Environments

`angular.json` defines named **configurations** (e.g. `production`, `staging`) that set optimization flags and **file replacements** (swapping `environment.ts` for `environment.prod.ts`). This is how per-environment settings (API URLs, feature flags) are baked in at build time.

```jsonc
"configurations": {
  "production": {
    "fileReplacements": [{ "replace": "src/environments/environment.ts",
                           "with": "src/environments/environment.prod.ts" }],
    "budgets": [{ "type": "initial", "maximumError": "1mb" }]
  }
}
```

Note: environment values are compiled into the client bundle, so they are **public** — never put secrets there.

## Budgets

**Bundle budgets** in `angular.json` warn or fail the build when bundles exceed size thresholds (initial bundle, component styles, etc.). They guard against gradual bundle bloat and keep load performance in check as the app grows.

## Bundle Analysis

To find what's inflating a bundle, generate stats and analyze them:

```bash
ng build --stats-json
npx source-map-explorer dist/**/main*.js     # or use the esbuild metafile
```

This reveals large dependencies, accidental eager imports of lazy features, and duplication — the starting point for size optimization.

## Source Maps and Debugging

Source maps map minified output back to source for debugging. They're on for development and can be enabled for production (`"sourceMap": true`) to debug prod issues or feed error-monitoring tools (Sentry), while typically not being served publicly.

## Deployment Targets

An Angular app builds to **static assets** (HTML/JS/CSS) that can be hosted anywhere:

* **Static hosts / CDNs** — Netlify, Vercel, Firebase Hosting, S3+CloudFront, GitHub Pages.
* **Web servers** — Nginx/Apache serving `dist/`.
* **Docker** — a multi-stage image (build with Node, serve with Nginx).
* **SSR** — requires a Node server (or serverless functions) to run the render (see SSR & Hydration).

## SPA Routing and `base href`

For client-side routing with the default `PathLocationStrategy`, the server must serve `index.html` for unknown deep-link paths (a **fallback rewrite**), or refreshing `/users/1` 404s. Set the correct `--base-href` when the app is served from a subpath. `HashLocationStrategy` (`#/route`) avoids the rewrite requirement but produces uglier URLs.

## CI/CD and Upgrades

Automate `npm ci` → `ng lint` → `ng test` (headless) → `ng build` in CI, then deploy the artifact. Keep the framework current with **`ng update`**, which runs migration schematics to apply breaking-change codemods automatically — a key reason Angular upgrades are relatively smooth.

## Interview Q&A

**Q: What build system does modern Angular use?**
A: The application builder (`@angular/build`) using esbuild for bundling and Vite for the dev server, replacing the older webpack-based builder for much faster builds.

**Q: What is AOT and why is it the production default?**
A: Ahead-of-Time compilation compiles templates/components at build time, giving faster startup (no shipped compiler), smaller bundles, and build-time template error detection.

**Q: What optimizations does a production build apply?**
A: Tree shaking, minification/mangling, bundling with code splitting for lazy chunks, and content hashing for cache busting — all automatic with `ng build`.

**Q: How do you manage per-environment configuration?**
A: Named configurations in `angular.json` with file replacements (`environment.ts` → `environment.prod.ts`); values are compiled into the bundle, so they're public — no secrets.

**Q: What are bundle budgets?**
A: Size thresholds in `angular.json` that warn or fail the build when bundles get too large, preventing gradual performance regressions.

**Q: How do you analyze bundle size?**
A: Build with `--stats-json` and inspect with `source-map-explorer` or the esbuild metafile to find large deps, duplication, and mis-bundled lazy features.

**Q: Where can you deploy an Angular app?**
A: As static assets on CDNs/static hosts, web servers, or Docker; SSR apps additionally need a Node/serverless server to render.

**Q: Why must the server fall back to `index.html`?**
A: With path-based routing, deep-link refreshes hit the server; it must serve `index.html` so Angular's router can handle the route, otherwise you get 404s. `--base-href` handles subpath hosting.

**Q: How do Angular upgrades stay manageable?**
A: `ng update` runs migration schematics that automatically apply breaking-change codemods, and the CLI/framework are versioned together.

**Q: Should production source maps be enabled?**
A: Optionally — they aid debugging and error monitoring (Sentry) but are usually generated for tooling rather than served publicly.

## Interview Notes

* Know the esbuild/Vite application builder and `ng build`/`serve`.
* Explain AOT benefits and that it's the production default.
* List automatic prod optimizations (tree shaking, minify, code split, hashing).
* Configure environments via `angular.json` file replacements; never ship secrets.
* Use budgets and bundle analysis to control size.
* Handle SPA routing fallback/`base href` and automate CI/CD; upgrade with `ng update`.
