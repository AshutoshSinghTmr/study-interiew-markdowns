# Angular Testing

## The Testing Landscape

Angular ships first-class testing support built on **`TestBed`** (a testing module that configures a mini Angular environment) plus a test runner. The default runner has historically been **Jasmine + Karma**; modern Angular is moving to alternatives like **Jest** and the **Web Test Runner** (Karma is deprecated). Tests fall into unit tests (services, pipes, pure logic), component tests (template + class), and **e2e** tests (Cypress/Playwright) that drive the real app.

## `TestBed` Basics

`TestBed` configures a testing module — declaring the component under test, its imports, and provider overrides — then creates a component fixture you can inspect and interact with.

```ts
beforeEach(() => TestBed.configureTestingModule({
  imports: [UserCardComponent],                 // standalone component
  providers: [{ provide: UserApi, useValue: fakeApi }],
}));

it('shows the user name', () => {
  const fixture = TestBed.createComponent(UserCardComponent);
  fixture.componentRef.setInput('name', 'Ada');  // set a signal/decorator input
  fixture.detectChanges();                        // run change detection
  const el = fixture.nativeElement as HTMLElement;
  expect(el.querySelector('h2')?.textContent).toContain('Ada');
});
```

`fixture.detectChanges()` triggers change detection (and lifecycle hooks) so the template reflects current state; `fixture.debugElement` offers a framework-level query API (`By.css`, `By.directive`).

## Testing Components

Component tests verify the rendered DOM and interaction behaviour:

* set inputs (`setInput`), call methods, dispatch DOM events;
* call `detectChanges()` and assert on the DOM;
* for async updates, use `fakeAsync` + `tick()` (virtual time) or `await fixture.whenStable()`.

```ts
it('emits on click', () => {
  const fixture = TestBed.createComponent(ButtonComponent);
  const spy = jasmine.createSpy();
  fixture.componentInstance.pressed.subscribe(spy);
  fixture.nativeElement.querySelector('button').click();
  expect(spy).toHaveBeenCalled();
});
```

## Testing Services

Services are the easiest to test — often plain unit tests with mocked dependencies. Inject via `TestBed.inject(Service)` (to exercise DI) or instantiate directly with fakes.

```ts
it('adds to cart', () => {
  const store = TestBed.inject(CartService);
  store.add({ id: 1, price: 10 } as Item);
  expect(store.total()).toBe(10);              // read a signal directly
});
```

## Testing HTTP

Use `provideHttpClientTesting` and `HttpTestingController` to assert outgoing requests and flush fake responses without hitting the network.

```ts
TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
const ctrl = TestBed.inject(HttpTestingController);

service.getUsers().subscribe(users => expect(users.length).toBe(1));
ctrl.expectOne('/api/users').flush([{ id: 1 }]);
ctrl.verify();                                 // asserts no unexpected/outstanding requests
```

## Mocking and Spies

Isolate the unit under test by replacing dependencies with fakes/spies (`jasmine.createSpyObj`, `jest.fn`) provided via `useValue`/`useClass`. Override component template dependencies with stub components when needed. Prefer testing behaviour through the public API over internal implementation details.

## Component Harnesses (CDK)

The **Component Test Harness** API (`@angular/cdk/testing`) provides stable, implementation-agnostic accessors for components (Angular Material and custom). Tests interact through the harness rather than brittle CSS selectors, so refactors don't break tests.

```ts
const harness = await TestbedHarnessEnvironment.loader(fixture).getHarness(MatButtonHarness);
await harness.click();
```

## Testing Signals and RxJS

* **Signals** — set inputs/signals, call `detectChanges()`, and assert on `signal()` values or the rendered DOM; computed values are read directly.
* **RxJS** — assert emitted values by subscribing, or use **marble testing** (`TestScheduler`) for time-based operators to verify emission timing deterministically.

## End-to-End Testing

E2E tests exercise the whole app in a real browser. Angular's original Protractor is deprecated; teams use **Cypress** or **Playwright**. E2E is slower and more brittle than unit/component tests, so follow the testing pyramid: many fast unit/component tests, fewer e2e tests covering critical user journeys.

## Best Practices

* Favour component/service tests over heavy e2e; keep the pyramid.
* Test public behaviour and the DOM users see, not private internals.
* Mock external dependencies (HTTP, time) for fast, deterministic tests.
* Use harnesses/`data-testid` over fragile CSS selectors.
* Prefer `fakeAsync`/`tick` for deterministic async control.

## Interview Q&A

**Q: What is `TestBed`?**
A: Angular's testing utility that configures a testing module (declarations, imports, provider overrides) and creates component fixtures, giving a controlled Angular environment for tests.

**Q: What does `fixture.detectChanges()` do?**
A: Runs change detection (and relevant lifecycle hooks) so the template reflects the current component state before you assert on the DOM.

**Q: How do you test components that use inputs?**
A: Create the fixture, set inputs with `fixture.componentRef.setInput(...)`, call `detectChanges()`, then assert on the rendered DOM or emitted outputs.

**Q: How do you test HTTP calls?**
A: With `provideHttpClientTesting` and `HttpTestingController` — `expectOne` the request, `flush` a fake response, and `verify()` there are no outstanding requests, avoiding real network calls.

**Q: How do you handle async in component tests?**
A: Use `fakeAsync` with `tick()` to control virtual time deterministically, or `await fixture.whenStable()` for real async completion.

**Q: What are component harnesses?**
A: CDK-provided, implementation-agnostic accessors (`@angular/cdk/testing`) for interacting with components in tests without brittle CSS selectors, so tests survive refactors.

**Q: How do you isolate a unit under test?**
A: Replace its dependencies with spies/fakes via `useValue`/`useClass`, and stub child components/templates as needed, testing behaviour through the public API.

**Q: How do you test time-based RxJS logic?**
A: Marble testing with `TestScheduler` to assert emission values and timing deterministically.

**Q: What's the state of e2e testing in Angular?**
A: Protractor is deprecated; use Cypress or Playwright. Keep e2e tests few and focused on critical journeys, per the testing pyramid.

**Q: `TestBed.inject` vs direct instantiation for services?**
A: `TestBed.inject` exercises DI and provider config; direct `new` with fakes is simpler for pure logic. Choose based on whether DI wiring is part of the test.

## Interview Notes

* Explain `TestBed`, fixtures, and `detectChanges()`.
* Test components via inputs/DOM/outputs; services via injection and public API.
* Use `HttpTestingController` for HTTP and `fakeAsync`/`tick` for async.
* Prefer harnesses/`data-testid` over brittle selectors.
* Know marble testing for RxJS and the testing pyramid (Cypress/Playwright for e2e).
* Note Karma/Protractor deprecation and the move to Jest/Web Test Runner.
