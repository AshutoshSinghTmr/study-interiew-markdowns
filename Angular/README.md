# Angular Interview Preparation

A standalone, revision-ready knowledge base for Angular interviews. Each file is a self-contained topic in a consistent format:

* concise, internals-aware prose with focused code examples;
* an **Interview Q&A** section of rapid question → answer flashcards for active recall;
* an **Interview Notes** section listing the key points to be able to discuss.

**Target version:** Angular v19/v20+ (standalone-first, signals, the `@if`/`@for`/`@switch`/`@defer` control flow, zoneless change detection). Legacy `NgModule`/decorator-based equivalents are called out where they still come up in interviews.

## Table of Contents

- [How to Revise](#how-to-revise)
- [Topics](#topics)
  - [Core & Components](#core--components)
  - [Dependency Injection & Services](#dependency-injection--services)
  - [Reactivity & Data](#reactivity--data)
  - [Routing & Forms](#routing--forms)
  - [State & Change Detection](#state--change-detection)
  - [Performance & Quality](#performance--quality)
  - [Rendering & Delivery](#rendering--delivery)

## How to Revise

1. Read a topic top-to-bottom once for understanding.
2. On later passes, cover the answers and drill the **Interview Q&A** flashcards.
3. Use the **Interview Notes** as a final pre-interview checklist per topic.

## Topics

### Core & Components

| Topic | Focus |
| --- | --- |
| [Overview and Architecture](Overview_and_Architecture.md) | Building blocks, CLI, workspace structure, standalone vs `NgModule`, bootstrapping, `ApplicationConfig` |
| [Components and Templates](Components_and_Templates.md) | `@Component`, lifecycle hooks, data binding, `@if`/`@for`/`@switch`/`@defer`, view encapsulation |
| [Component Communication](Component_Communication.md) | `@Input`/`@Output`, signal `input()`/`output()`/`model()`, view/content queries, content projection |
| [Directives and Pipes](Directives_and_Pipes.md) | Attribute vs structural directives, `HostBinding`/`HostListener`, host directives, pure vs impure pipes |

### Dependency Injection & Services

| Topic | Focus |
| --- | --- |
| [Dependency Injection](Dependency_Injection.md) | Hierarchical injectors, providers, `inject()`, `InjectionToken`, resolution modifiers, tree-shaking |
| [Services and Providers](Services_and_Providers.md) | Service pattern, singleton vs scoped, stateful services, facades, `provideX` provider functions |

### Reactivity & Data

| Topic | Focus |
| --- | --- |
| [Signals and Reactivity](Signals_and_Reactivity.md) | `signal`/`computed`/`effect`, signal I/O, `toSignal`/`toObservable`, `resource`, signals vs RxJS |
| [RxJS and Observables](RxJS_and_Observables.md) | Observable vs Promise, Subjects, flattening operators, `async` pipe, subscription management |
| [HTTP Client and Interceptors](HTTP_Client_and_Interceptors.md) | `provideHttpClient`, typed requests, functional interceptors, error/retry, `HttpTestingController` |

### Routing & Forms

| Topic | Focus |
| --- | --- |
| [Routing and Navigation](Routing_and_Navigation.md) | `provideRouter`, params, lazy loading, functional guards/resolvers, input binding, preloading |
| [Forms and Validation](Forms_and_Validation.md) | Template-driven vs reactive, typed forms, custom/async/cross-field validators, `ControlValueAccessor` |

### State & Change Detection

| Topic | Focus |
| --- | --- |
| [State Management](State_Management.md) | Local vs store, NgRx (actions/reducers/selectors/effects), SignalStore, ComponentStore, choosing |
| [Change Detection and Zoneless](Change_Detection_and_Zoneless.md) | zone.js, `OnPush`, `ChangeDetectorRef`, signals + CD, zoneless, `ExpressionChanged...` error |

### Performance & Quality

| Topic | Focus |
| --- | --- |
| [Performance Optimization](Performance_Optimization.md) | Lazy loading, `@defer`, `OnPush` + signals, `@for` track, `NgOptimizedImage`, budgets, virtual scroll |
| [Testing](Testing.md) | `TestBed`, component/service testing, `HttpTestingController`, CDK harnesses, Jasmine/Jest, e2e |
| [Security](Security.md) | XSS auto-escaping/sanitization, `DomSanitizer` risks, Trusted Types/CSP, XSRF, safe URLs |

### Rendering & Delivery

| Topic | Focus |
| --- | --- |
| [Server-Side Rendering and Hydration](Server_Side_Rendering_and_Hydration.md) | `@angular/ssr`, hydration (incremental), `TransferState`, `isPlatformBrowser`, SSG, pitfalls |
| [Build and Deployment](Build_and_Deployment.md) | CLI application builder (esbuild/Vite), AOT, budgets, configurations, bundle analysis, hosting/CI |
| [Internationalization (i18n)](Internationalization_i18n.md) | Built-in `i18n`/`$localize`, extraction, XLIFF, `LOCALE_ID`, locale pipes, ICU, alternatives |
| [PWA and Service Workers](PWA_and_Service_Workers.md) | `@angular/pwa`, `ngsw` caching strategies, `SwUpdate`/`SwPush`, offline, update flow |
