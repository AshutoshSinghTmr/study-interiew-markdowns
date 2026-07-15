# Angular HTTP Client and Interceptors

## Setting Up `HttpClient`

Angular's `HttpClient` (from `@angular/common/http`) is a typed, observable-based HTTP API. In standalone apps you register it with `provideHttpClient` in `ApplicationConfig`.

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, errorInterceptor]),
      withFetch(),                          // use the fetch API backend
    ),
  ],
};
```

`withFetch()` switches to the fetch-based backend (recommended, SSR-friendly); `withInterceptors([...])` registers functional interceptors.

## Making Requests

Methods (`get`, `post`, `put`, `patch`, `delete`) return **cold** Observables — the request fires on subscription and is cancelled on unsubscribe. Use the generic type parameter for typed responses.

```ts
export class UserApi {
  private http = inject(HttpClient);

  getUsers(page: number) {
    const params = new HttpParams().set('page', page);
    return this.http.get<User[]>('/api/users', { params });
  }

  create(user: NewUser) {
    return this.http.post<User>('/api/users', user);
  }
}
```

Pass options for `params` (`HttpParams`), `headers` (`HttpHeaders`), `responseType`, `withCredentials`, and `observe: 'response'` (to get the full `HttpResponse` including status/headers) or `observe: 'events'` (for upload/download progress).

## Consuming Responses

Subscribe via the `async` pipe, `toSignal`, or an explicit subscription with cleanup. For one-shot calls in imperative code, convert to a Promise with `firstValueFrom`.

```ts
users = toSignal(this.userApi.getUsers(1), { initialValue: [] });

async load() {
  const user = await firstValueFrom(this.userApi.create(dto));
}
```

Because requests are cold, subscribing twice sends two requests; use `shareReplay(1)` to cache/multicast a shared response.

## Functional Interceptors

Interceptors sit in a chain that every request/response passes through — ideal for cross-cutting concerns (auth, logging, error handling, retries). Modern Angular uses **functional interceptors** (`HttpInterceptorFn`) registered via `withInterceptors`.

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(authReq);
};
```

Requests are **immutable** — you must `req.clone(...)` to modify them. `next(req)` forwards to the next interceptor/backend and returns the response stream.

## Error Handling and Retry Interceptors

An error interceptor centralizes handling of `HttpErrorResponse` (status codes, network failures) and retry logic.

```ts
export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    retry({ count: 2, delay: 1000 }),
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) inject(AuthService).logout();
      inject(ToastService).show(err.message);
      return throwError(() => err);
    }),
  );
```

`HttpErrorResponse` distinguishes client/network errors (`error` is an `ErrorEvent`/object, `status` 0) from server errors (`status >= 400`).

## `HttpContext`

`HttpContext` passes per-request metadata to interceptors via typed tokens (e.g. "skip auth for this call", "show a loader"), without abusing headers.

```ts
const SKIP_AUTH = new HttpContextToken(() => false);
this.http.get('/public', { context: new HttpContext().set(SKIP_AUTH, true) });
```

## XSRF/CSRF Protection

`HttpClient` has built-in XSRF support: it reads a cookie (default `XSRF-TOKEN`) and sends it as a header (`X-XSRF-TOKEN`) on mutating requests, configurable via `withXsrfConfiguration`. This mitigates cross-site request forgery for cookie-authenticated APIs (see Security).

## Testing HTTP

Use `provideHttpClientTesting` and `HttpTestingController` to assert requests and flush fake responses — no real network.

```ts
const ctrl = TestBed.inject(HttpTestingController);
service.getUsers(1).subscribe(res => expect(res.length).toBe(2));
ctrl.expectOne('/api/users?page=1').flush([{ id: 1 }, { id: 2 }]);
ctrl.verify();
```

## Interview Q&A

**Q: How do you set up `HttpClient` in a standalone app?**
A: Add `provideHttpClient()` (often with `withFetch()` and `withInterceptors([...])`) to the `ApplicationConfig` providers.

**Q: Are `HttpClient` requests hot or cold?**
A: Cold — the request only fires on subscription and is cancelled on unsubscribe; subscribing multiple times sends multiple requests. Use `shareReplay` to share one response.

**Q: What is an HTTP interceptor and what is it used for?**
A: A function in the request/response chain that handles cross-cutting concerns — attaching auth tokens, logging, retries, error handling, and loaders — without touching individual calls.

**Q: Why must you clone the request in an interceptor?**
A: `HttpRequest` objects are immutable; to add headers/params you create a modified copy with `req.clone(...)` and pass it to `next`.

**Q: How do you handle errors globally?**
A: With an error interceptor using `catchError` on `next(req)`, inspecting `HttpErrorResponse.status`, applying retries, redirecting on 401, and rethrowing or returning a fallback.

**Q: How do you get the full response or track progress?**
A: Use `observe: 'response'` for the full `HttpResponse` (status/headers) or `observe: 'events'` with `reportProgress: true` for upload/download progress events.

**Q: What is `HttpContext` for?**
A: Passing typed per-request metadata to interceptors (e.g. skip auth, show loader) without misusing headers.

**Q: How does Angular help with CSRF?**
A: `HttpClient` reads an XSRF cookie and automatically adds it as a header on mutating requests, configurable via `withXsrfConfiguration`.

**Q: How do you test HTTP calls?**
A: With `provideHttpClientTesting` and `HttpTestingController` — expect a request (`expectOne`), `flush` a fake response, and `verify()` no outstanding requests.

**Q: How do you turn a request into a Promise?**
A: `firstValueFrom(obs$)` (or `lastValueFrom`) from RxJS, useful in async/await imperative flows.

## Interview Notes

* Set up with `provideHttpClient` (`withFetch`, `withInterceptors`) and use typed generics.
* Stress requests are cold; use `shareReplay` to avoid duplicate calls.
* Explain functional interceptors, request immutability/`clone`, and the chain.
* Describe global error handling/retry and `HttpErrorResponse` client vs server errors.
* Mention `HttpContext`, built-in XSRF, and `observe` options for response/progress.
* Know `HttpTestingController` for testing.
