# Angular Forms and Validation

## Two Forms Approaches

Angular offers two form-building strategies:

* **Template-driven forms** — logic lives in the template using `ngModel`; Angular builds the model implicitly. Simple and quick for small forms, but harder to test and scale.
* **Reactive forms** — the form model is defined explicitly in the component class with `FormControl`/`FormGroup`/`FormArray`. Type-safe, testable, and scalable; the recommended approach for anything non-trivial.

Both are built on the same base classes (`AbstractControl`); the difference is where the model is defined.

## Template-Driven Forms

Import `FormsModule`, bind with `[(ngModel)]`, and read validity from a template reference to the `NgForm`/`NgModel`.

```html
<form #f="ngForm" (ngSubmit)="save(f.value)">
  <input name="email" ngModel required email #email="ngModel" />
  @if (email.invalid && email.touched) { <span>Invalid email</span> }
  <button [disabled]="f.invalid">Save</button>
</form>
```

Validation is expressed as directives (`required`, `email`, `minlength`) on the inputs. The model updates asynchronously, which is why testing is trickier.

## Reactive Forms

Import `ReactiveFormsModule` and build the model in code, usually with `FormBuilder`. Bind with `[formGroup]` and `formControlName`.

```ts
private fb = inject(FormBuilder);
form = this.fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(8)]],
});

save() {
  if (this.form.valid) this.api.login(this.form.getRawValue());
}
```

```html
<form [formGroup]="form" (ngSubmit)="save()">
  <input formControlName="email" />
  <input formControlName="password" type="password" />
  <button [disabled]="form.invalid">Log in</button>
</form>
```

## Typed Reactive Forms

Since Angular 14, reactive forms are **strictly typed**: `FormGroup` infers a typed value, and controls know their value type. Use `NonNullableFormBuilder` (or `nonNullable: true`) so controls reset to their initial value rather than `null`.

```ts
form = inject(NonNullableFormBuilder).group({
  name: ['', Validators.required],       // FormControl<string>
  age: [0],                              // FormControl<number>
});
// form.value is { name?: string; age?: number }; getRawValue() includes disabled controls
```

## FormControl, FormGroup, FormArray

* **`FormControl`** — a single field's value, validity, and state.
* **`FormGroup`** — a fixed set of named controls (an object).
* **`FormArray`** — a dynamic list of controls (add/remove rows).

```ts
form = this.fb.group({
  name: [''],
  phones: this.fb.array([this.fb.control('')]),   // dynamic list
});
get phones() { return this.form.get('phones') as FormArray; }
addPhone() { this.phones.push(this.fb.control('')); }
```

## Validators

* **Built-in** — `Validators.required`, `min`, `max`, `minLength`, `maxLength`, `pattern`, `email`.
* **Custom synchronous** — a function returning `ValidationErrors | null`.
* **Cross-field** — a validator on the group comparing multiple controls.
* **Async** — returns an `Observable`/`Promise` of errors (e.g. server-side uniqueness check).

```ts
// custom sync
export const noSpaces: ValidatorFn = c =>
  /\s/.test(c.value ?? '') ? { spaces: true } : null;

// cross-field (on the group)
export const passwordsMatch: ValidatorFn = g =>
  g.get('pw')!.value === g.get('confirm')!.value ? null : { mismatch: true };

// async uniqueness
export function uniqueEmail(api: UserApi): AsyncValidatorFn {
  return c => api.exists(c.value).pipe(map(taken => taken ? { taken: true } : null));
}
```

Attach with `this.fb.group({...}, { validators: passwordsMatch })` and read errors via `control.errors` / `control.hasError('taken')`.

## Control State and Value Changes

Controls expose reactive state: `valid`/`invalid`, `touched`/`untouched`, `dirty`/`pristine`, `pending`, and `disabled`. Streams `valueChanges` and `statusChanges` let you react (e.g. dependent fields, autosave). `updateOn: 'blur' | 'submit'` controls when validation runs.

```ts
this.form.controls.country.valueChanges
  .pipe(takeUntilDestroyed())
  .subscribe(country => this.loadStates(country));
```

## Custom Form Controls: `ControlValueAccessor`

To make your own component usable with `formControlName`/`ngModel`, implement **`ControlValueAccessor`** — the bridge between the form API and your component's view. Implement `writeValue`, `registerOnChange`, `registerOnTouched`, and `setDisabledState`, and register the component as an `NG_VALUE_ACCESSOR`.

```ts
@Component({
  selector: 'app-rating',
  providers: [{ provide: NG_VALUE_ACCESSOR, useExisting: RatingComponent, multi: true }],
  template: `…stars…`,
})
export class RatingComponent implements ControlValueAccessor {
  value = signal(0);
  onChange: (v: number) => void = () => {};
  writeValue(v: number) { this.value.set(v ?? 0); }
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) {}
  select(v: number) { this.value.set(v); this.onChange(v); }
}
```

## Interview Q&A

**Q: Template-driven vs reactive forms?**
A: Template-driven defines the model implicitly via `ngModel` in the template — simple but less testable. Reactive defines an explicit, typed model in code (`FormGroup`/`FormControl`) — scalable, testable, and preferred for complex forms.

**Q: What are `FormControl`, `FormGroup`, and `FormArray`?**
A: A control is a single field; a group is a fixed named set of controls; an array is a dynamic list of controls you can add/remove at runtime.

**Q: What are typed reactive forms and `NonNullableFormBuilder`?**
A: Since v14 form values are strictly typed. `NonNullableFormBuilder` (or `nonNullable: true`) makes controls reset to their initial value instead of `null`, giving cleaner types.

**Q: How do you write a custom validator?**
A: A `ValidatorFn` returning `ValidationErrors | null`. Put it on a control for field validation or on a group for cross-field checks; async validators return an observable/promise of errors.

**Q: How do cross-field validators work?**
A: Attach a validator to the `FormGroup` (via the group options) so it can compare multiple child controls, e.g. password/confirm matching, and set a group-level error.

**Q: What is `ControlValueAccessor`?**
A: The interface that lets a custom component integrate with Angular forms; implementing `writeValue`/`registerOnChange`/`registerOnTouched` and registering `NG_VALUE_ACCESSOR` makes it work with `formControlName`/`ngModel`.

**Q: How do you react to value changes?**
A: Subscribe to `control.valueChanges`/`statusChanges` (with cleanup like `takeUntilDestroyed`) for dependent fields, autosave, or conditional validation.

**Q: What is `updateOn`?**
A: A control/group option (`'change' | 'blur' | 'submit'`) controlling when values and validation update — useful to reduce validation churn.

**Q: `value` vs `getRawValue()`?**
A: `value` omits disabled controls; `getRawValue()` returns all controls including disabled ones — important when submitting complete form data.

**Q: How do you show validation messages appropriately?**
A: Check control state like `control.invalid && (control.touched || control.dirty)` and read specific errors via `control.hasError('required')`.

## Interview Notes

* Contrast template-driven (implicit, `ngModel`) vs reactive (explicit, typed) and when to use each.
* Know `FormControl`/`FormGroup`/`FormArray` and typed forms with `NonNullableFormBuilder`.
* Write built-in, custom, cross-field (group-level), and async validators.
* Explain control state flags, `valueChanges`/`statusChanges`, and `updateOn`.
* Implement `ControlValueAccessor` for custom form components.
* Remember `getRawValue()` includes disabled controls.
