# Angular Internationalization (i18n)

## What i18n Covers

Internationalization (**i18n**) prepares an app to support multiple languages and locales; **localization (l10n)** is producing a specific translation. Angular has a first-party i18n system plus locale-aware pipes, and there are popular community alternatives for runtime translation.

## Built-in i18n: Marking Text

Angular's built-in i18n marks translatable content in templates with the **`i18n`** attribute; attributes are translated with `i18n-<attr>`.

```html
<h1 i18n="site header|Main greeting@@homeTitle">Welcome</h1>
<img [src]="logo" i18n-alt alt="Company logo" />
```

The `i18n` value can carry `meaning|description@@customId` metadata: a **meaning** (disambiguates identical text), a **description** (helps translators), and a stable **custom ID** (`@@`) so translations survive text edits.

## `$localize` in Code

For strings outside templates, the **`$localize`** tagged template literal marks translatable text in TypeScript.

```ts
const msg = $localize`:@@savedMsg:Your changes were saved`;
```

`$localize` is the runtime primitive underpinning Angular i18n and enables both compile-time and (with tooling) runtime translation.

## Extraction and Translation Files

Run extraction to collect all marked messages into a translation source file:

```bash
ng extract-i18n --output-path src/locale     # produces messages.xlf
```

Angular uses **XLIFF** (`.xlf`, versions 1.2/2.0) by default; XMB/JSON are also supported. Translators produce per-locale files (`messages.fr.xlf`, `messages.de.xlf`) with the translated `<target>` for each `<source>`.

## Build-Time Translation (Angular's Model)

Angular's native i18n is **build-time**: the CLI produces a **separate, fully-translated build per locale**, each with translations baked in (no runtime translation cost). You configure locales in `angular.json` and build all of them; each is deployed under its own path/domain.

```jsonc
"i18n": {
  "sourceLocale": "en-US",
  "locales": { "fr": "src/locale/messages.fr.xlf", "de": "src/locale/messages.de.xlf" }
}
```

Pros: no runtime overhead, optimal bundle per language. Cons: switching language means loading a different build (usually a redirect), and N builds to deploy.

## `LOCALE_ID` and Locale Data

The **`LOCALE_ID`** token tells Angular the active locale, driving locale-aware pipes. Locale data (date/number/currency formats, plural rules) is registered via `registerLocaleData`. Each localized build sets its own `LOCALE_ID`.

```ts
providers: [{ provide: LOCALE_ID, useValue: 'fr' }]
```

## Locale-Aware Pipes

The formatting pipes automatically adapt to the active locale:

```html
{{ price | currency:'EUR' }}      <!-- 1 234,56 € in fr, €1,234.56 in en -->
{{ today | date:'long' }}
{{ ratio | percent }}
{{ count | number:'1.0-2' }}
```

These handle locale differences (decimal separators, date order, currency symbols) so you don't format manually.

## Pluralization and Select (ICU)

Angular supports **ICU message format** for plurals and gender/selection, so translations handle count-dependent and choice-dependent wording correctly.

```html
<span i18n>{count, plural, =0 {no items} =1 {one item} other {{{count}} items}}</span>
<span i18n>{gender, select, male {he} female {she} other {they}}</span>
```

## RTL and Layout

Localization includes **right-to-left (RTL)** languages (Arabic, Hebrew). Set `dir="rtl"` on the document/root and use CSS logical properties (`margin-inline-start`, `text-align: start`) so layouts mirror correctly without duplicating styles.

## Runtime Alternatives (ngx-translate / Transloco)

Because Angular's native i18n is build-time, apps needing **runtime language switching** without separate builds often use community libraries: **`@ngx-translate/core`** or **Transloco**. These load JSON translation files at runtime and switch languages instantly via a pipe/directive/service, at the cost of a small runtime overhead. Choose native i18n for max performance/SEO with per-locale deployments, or a runtime library for instant in-app switching.

## Interview Q&A

**Q: How do you mark text for translation in Angular?**
A: With the `i18n` attribute on elements (`i18n-<attr>` for attributes) in templates, and the `$localize` tagged template literal for strings in TypeScript.

**Q: What metadata can the `i18n` attribute carry?**
A: `meaning|description@@customId` — a meaning to disambiguate identical text, a description for translators, and a stable custom ID so translations persist across text edits.

**Q: How does Angular's native i18n build work?**
A: It's build-time: `ng extract-i18n` collects messages (XLIFF), translators fill per-locale files, and the CLI produces a separate fully-translated build per locale with no runtime translation cost.

**Q: What is `LOCALE_ID`?**
A: A DI token identifying the active locale that drives locale-aware pipes; each localized build provides its own value, with locale data registered via `registerLocaleData`.

**Q: How do locale-aware pipes help?**
A: `date`, `currency`, `number`, and `percent` format values according to the active locale (separators, symbols, ordering), avoiding manual formatting.

**Q: How does Angular handle plurals?**
A: Via ICU message format (`{count, plural, =0 {...} other {...}}`) and `select` for choices, so translated wording adapts to counts and categories.

**Q: Native i18n vs ngx-translate/Transloco?**
A: Native i18n is build-time (best performance/SEO, one build per locale, redirect to switch); ngx-translate/Transloco load JSON at runtime for instant in-app language switching with some overhead.

**Q: What translation file format does Angular use?**
A: XLIFF (1.2/2.0) by default, with XMB/JSON also supported; each locale has its own file with translated targets.

**Q: How do you support RTL languages?**
A: Set `dir="rtl"` and use CSS logical properties so layout mirrors automatically, in addition to translating text.

**Q: Can you put secrets or environment-specific config in translations?**
A: No — translation files ship to the client like the rest of the bundle; they're for display text only.

## Interview Notes

* Mark text with `i18n`/`i18n-<attr>` and `$localize`; use `meaning|description@@id` metadata.
* Explain the extract → translate (XLIFF) → build-time per-locale build flow.
* Know `LOCALE_ID`, `registerLocaleData`, and locale-aware pipes.
* Handle plurals/selection with ICU and RTL with logical CSS.
* Contrast native build-time i18n with runtime ngx-translate/Transloco and when each fits.
