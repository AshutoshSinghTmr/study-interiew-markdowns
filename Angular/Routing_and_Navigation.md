# Angular Routing and Navigation

## Configuring the Router

The Angular Router maps URLs to components, enabling a single-page app with multiple views. In a standalone app you configure it with `provideRouter(routes)` and render matched components in a `<router-outlet>`.

```ts
export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'users/:id', component: UserComponent },
  { path: 'admin', loadChildren: () => import('./admin/routes').then(m => m.ADMIN_ROUTES) },
  { path: '**', component: NotFoundComponent },   // wildcard (order matters)
];

provideRouter(routes, withComponentInputBinding(), withViewTransitions());
```

Routes are matched **top-to-bottom, first-match-wins**, so specific routes precede the `**` wildcard.

## Navigation

Navigate declaratively with `routerLink` or imperatively with the `Router`.

```html
<a [routerLink]="['/users', user.id]" routerLinkActive="active">{{ user.name }}</a>
```

```ts
private router = inject(Router);
this.router.navigate(['/users', id], { queryParams: { tab: 'profile' } });
```

`routerLinkActive` toggles a class when the link's route is active — the idiomatic way to highlight the current nav item.

## Reading Route Parameters

Access params, query params, and data via `ActivatedRoute` (observables or snapshot). The modern approach is **component input binding** (`withComponentInputBinding`), which binds route params/query/data directly to component inputs.

```ts
// With withComponentInputBinding(): route param :id maps to an input
export class UserComponent {
  id = input.required<string>();          // bound from /users/:id
}
```

```ts
// Classic: observable params (reacts to param changes without recreating the component)
private route = inject(ActivatedRoute);
user$ = this.route.paramMap.pipe(switchMap(p => this.api.getUser(p.get('id')!)));
```

Use the observable/`paramMap` form when the same component instance is reused across param changes (e.g. navigating `/users/1` → `/users/2`), since Angular won't recreate it.

## Lazy Loading

Lazy loading defers loading a feature's code until its route is visited, shrinking the initial bundle. Use `loadComponent` for a single standalone component or `loadChildren` for a set of routes.

```ts
{ path: 'reports', loadComponent: () => import('./reports.component').then(m => m.ReportsComponent) },
{ path: 'admin', loadChildren: () => import('./admin/routes').then(m => m.ADMIN_ROUTES) },
```

Lazy-loaded routes create their own environment injector, so route-level `providers` are scoped to that feature.

## Route Guards

Guards control navigation. Modern Angular uses **functional guards** returning `boolean | UrlTree | Observable/Promise` of those:

* `canActivate` — allow/deny entering a route.
* `canActivateChild` — guard child routes.
* `canDeactivate` — confirm leaving (e.g. unsaved changes).
* `canMatch` — decide whether a route matches at all (great for feature flags / role-based lazy loading, avoiding even loading the chunk).

```ts
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  return auth.isLoggedIn() ? true : inject(Router).createUrlTree(['/login']);
};

// route: { path: 'admin', canMatch: [authGuard], loadChildren: ... }
```

Returning a `UrlTree` redirects instead of just blocking.

## Resolvers

A **resolver** pre-fetches data before the route activates, so the component renders with data ready (no empty flash). Functional resolvers return the data (or an observable of it).

```ts
export const userResolver: ResolveFn<User> = (route) =>
  inject(UserApi).getUser(route.paramMap.get('id')!);

// route: { path: 'users/:id', resolve: { user: userResolver }, component: UserComponent }
```

Access resolved data via `ActivatedRoute.data` or a bound input. Use resolvers judiciously — they delay navigation until data arrives.

## Child Routes and Nested Outlets

Routes can nest via a `children` array rendered in a child `<router-outlet>`, building master-detail and layout hierarchies. Route `data` attaches static metadata (titles, roles); `title` (with an optional `TitleStrategy`) sets the document title per route.

## Preloading and Advanced Features

Preloading strategies (`withPreloading(PreloadAllModules)` or a custom strategy) fetch lazy chunks in the background after initial load, balancing startup size with navigation speed. Other features: `withViewTransitions()` (native view transition animations), `withInMemoryScrolling()` (scroll position restoration), and route-level redirects (`redirectTo`).

## Interview Q&A

**Q: How do you configure routing in a standalone app?**
A: Call `provideRouter(routes)` in `ApplicationConfig` and place a `<router-outlet>` where matched components should render; define routes as a `Routes` array.

**Q: How are routes matched?**
A: Top-to-bottom, first match wins, so specific paths must precede the `**` wildcard; `pathMatch: 'full'` is used for exact (often empty-path) matches.

**Q: How do you read route parameters, and when use snapshot vs observable?**
A: Via `ActivatedRoute` (`paramMap`/`snapshot`) or component input binding. Use the observable form when the component instance is reused across param changes; snapshot is fine when it's recreated.

**Q: `loadComponent` vs `loadChildren`?**
A: `loadComponent` lazily loads a single standalone component; `loadChildren` lazily loads a set of child routes. Both defer code and create a scoped injector.

**Q: What guards exist and what does each do?**
A: `canActivate` (enter), `canActivateChild` (children), `canDeactivate` (leave), and `canMatch` (whether the route matches at all — ideal for role/feature-flag lazy loading). Returning a `UrlTree` redirects.

**Q: What is a resolver and its trade-off?**
A: A function that pre-fetches data before activation so the view renders ready; the trade-off is delayed navigation until the data resolves.

**Q: What is `withComponentInputBinding`?**
A: A router feature that binds route params, query params, and resolved data directly to component `@Input`/signal inputs, removing boilerplate `ActivatedRoute` wiring.

**Q: How does preloading work?**
A: After the initial app loads, a preloading strategy (e.g. `PreloadAllModules` or custom) fetches lazy route chunks in the background so later navigation is instant.

**Q: How do you redirect in a guard?**
A: Return a `UrlTree` (e.g. `Router.createUrlTree(['/login'])`) from the guard instead of `false`.

**Q: How do you set the page title per route?**
A: Use the route `title` property (optionally with a custom `TitleStrategy`) which the router applies on navigation.

## Interview Notes

* Configure via `provideRouter`, `<router-outlet>`, and a `Routes` array; know first-match-wins.
* Navigate with `routerLink`/`Router.navigate` and highlight via `routerLinkActive`.
* Read params through input binding or `ActivatedRoute`; know snapshot vs observable reuse.
* Explain lazy loading (`loadComponent`/`loadChildren`) and scoped route injectors.
* Enumerate guards (esp. `canMatch` for lazy role checks) and returning `UrlTree` to redirect.
* Describe resolvers and preloading strategies and their trade-offs.
