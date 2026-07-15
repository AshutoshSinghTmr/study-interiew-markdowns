# Angular Overview and Architecture

## What Angular Is

Angular is a comprehensive, opinionated **framework** (not just a library) for building client-side single-page applications with TypeScript. Unlike view-only libraries, it ships batteries-included: components, dependency injection, routing, forms, an HTTP client, testing utilities, and a powerful CLI — all versioned and upgraded together. This cohesion is its main trade-off versus React: less choice, more consistency.

## The Building Blocks

* **Components** — the fundamental UI unit: a TypeScript class with an HTML template and styles, driving a piece of the screen.
* **Templates** — declarative HTML with Angular binding syntax and control flow.
* **Directives** — add behaviour to elements (structural change the DOM layout, attribute change appearance/behaviour).
* **Services** — reusable, injectable logic (data access, state) shared via DI.
* **Dependency Injection** — the framework that provides services to the classes that need them.
* **Pipes** — declarative value transformers used in templates.

## Standalone Components vs NgModules

Historically every component/directive/pipe was declared in an **`NgModule`** (`@NgModule` with `declarations`, `imports`, `providers`). Modern Angular is **standalone-first**: components declare their own dependencies via `imports`, removing the `NgModule` boilerplate. Standalone is the default in new projects (Angular 17+); `NgModule` still appears in legacy code and some libraries.

```ts
@Component({
  selector: 'app-user',
  imports: [CommonModule, RouterLink],   // standalone: dependencies declared here
  template: `<a [routerLink]="['/users', id()]">{{ name() }}</a>`,
})
export class UserComponent {
  id = input.required<number>();
  name = input('');
}
```

## Bootstrapping the Application

A modern app boots from `main.ts` with `bootstrapApplication`, passing the root component and an `ApplicationConfig` of providers — instead of the legacy `platformBrowserDynamic().bootstrapModule(AppModule)`.

```ts
// main.ts
bootstrapApplication(AppComponent, appConfig);

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
    provideZonelessChangeDetection(),      // modern zoneless option
  ],
};
```

Provider functions (`provideRouter`, `provideHttpClient`, ...) are the standalone replacement for feature `NgModule`s and are tree-shakable.

## Workspace and Project Structure

The **Angular CLI** (`ng`) scaffolds and manages a **workspace** defined by `angular.json`. A workspace can hold multiple projects (apps and libraries) under `projects/`. Key files: `angular.json` (build/test config), `tsconfig.json` (TypeScript), `package.json` (deps), and `src/` with `main.ts`, `index.html`, and the app source. Common commands: `ng new`, `ng generate` (`g c`/`g s`), `ng serve`, `ng build`, `ng test`, `ng lint`, and `ng update` for guided upgrades.

## How Rendering Works (High Level)

Angular compiles templates **ahead of time (AOT)** into efficient JavaScript instructions that create and update DOM nodes. At runtime, **change detection** synchronizes the component state with the DOM. Historically this was triggered by **zone.js** (which monkey-patches async APIs); modern Angular is moving to **signals** and **zoneless** change detection for fine-grained, more efficient updates (see Change Detection & Zoneless).

## TypeScript and Decorators

Angular is built on TypeScript: types, interfaces, generics, and **decorators** (`@Component`, `@Injectable`, `@Directive`, `@Pipe`) that attach metadata the compiler reads. Decorator metadata tells Angular how to compile and wire each class. Strong typing extends to templates (with strict template checking) and reactive forms.

## Angular vs React/Vue (Framing)

* **Framework vs library** — Angular provides routing, forms, HTTP, DI, and testing out of the box; React composes third-party libraries.
* **Language** — TypeScript-first with decorators and DI, versus React's JSX and hooks.
* **Reactivity** — historically zone.js-based CD; now signals (fine-grained) similar in spirit to Solid/Vue reactivity.
* **Opinionated structure** — CLI, style guide, and conventions promote consistency across large teams.

## Interview Q&A

**Q: Is Angular a library or a framework, and why does it matter?**
A: A framework — it includes routing, forms, HTTP, DI, and testing as first-party, versioned pieces. This gives consistency and guided upgrades at the cost of flexibility, unlike a view library like React.

**Q: What are standalone components and how do they differ from NgModules?**
A: Standalone components declare their own template dependencies via `imports` and don't need to be declared in an `NgModule`. They're the modern default, removing module boilerplate; `NgModule` remains for legacy code and some libraries.

**Q: How does a modern Angular app bootstrap?**
A: `main.ts` calls `bootstrapApplication(RootComponent, appConfig)`, where `appConfig` supplies providers via functions like `provideRouter` and `provideHttpClient` — replacing `bootstrapModule(AppModule)`.

**Q: What are the core building blocks of Angular?**
A: Components, templates, directives, services, dependency injection, and pipes — with routing and forms as major subsystems.

**Q: What does the Angular CLI do?**
A: Scaffolds and manages the workspace (`ng new`/`generate`), runs the dev server (`serve`), builds for production (`build`), runs tests/lint, and performs guided upgrades (`ng update`).

**Q: What is AOT compilation?**
A: Ahead-of-time compilation of templates and components into optimized JavaScript at build time, catching template errors early and improving startup versus runtime (JIT) compilation.

**Q: What role do decorators play?**
A: They attach metadata (`@Component`, `@Injectable`, etc.) that the Angular compiler reads to know how to compile templates, register providers, and wire dependencies.

**Q: What is `ApplicationConfig`?**
A: The object passed to `bootstrapApplication` holding the root-level `providers` (router, HTTP, change detection, etc.) — the standalone alternative to root `NgModule` providers.

**Q: How is Angular's reactivity evolving?**
A: From zone.js-driven change detection toward signals and zoneless change detection for fine-grained, more performant updates.

## Interview Notes

* Frame Angular as an opinionated, batteries-included framework and contrast with React.
* Explain standalone-first architecture and how it replaces `NgModule` boilerplate.
* Describe `bootstrapApplication` + `ApplicationConfig` + `provideX` functions.
* Know the CLI, workspace layout (`angular.json`), and core building blocks.
* Mention AOT compilation and the shift from zone.js to signals/zoneless.
