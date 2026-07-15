# Angular Security

## Angular's Security Posture

Angular is designed to be **secure by default**, with strong built-in protections against the most common web vulnerability, cross-site scripting (XSS). Understanding what the framework does automatically — and where you can accidentally disable it — is the core of Angular security.

## Cross-Site Scripting (XSS) and Auto-Escaping

XSS occurs when attacker-controlled data is interpreted as code (HTML/JS) in the browser. Angular defends by treating all values bound in templates as **untrusted data, not code**: interpolation and property bindings are **contextually escaped/sanitized** automatically.

```html
<p>{{ userInput }}</p>        <!-- always escaped: <script> renders as text -->
<div [innerHTML]="userHtml"></div>  <!-- sanitized: dangerous tags/attrs stripped -->
```

Angular's **sanitizer** cleans values for security contexts (HTML, style, URL, resource URL), removing scripts and dangerous attributes while keeping safe markup. Because binding, not string concatenation, is the norm, most XSS is prevented by design.

## The Danger of Bypassing Sanitization

The `DomSanitizer.bypassSecurityTrust*` methods (`bypassSecurityTrustHtml`, `...Url`, `...ResourceUrl`, `...Script`, `...Style`) tell Angular to trust a value and **skip sanitization**. They are a deliberate escape hatch and a serious risk: never pass user-controlled data to them.

```ts
// DANGEROUS if `raw` contains user input — reintroduces XSS
this.safeHtml = this.sanitizer.bypassSecurityTrustHtml(raw);
```

Only bypass for values you fully control (e.g. a trusted embed URL constant), and prefer never bypassing at all.

## `innerHTML` and Direct DOM Manipulation

Binding to `[innerHTML]` is sanitized, but building HTML strings from user input and injecting via native `element.innerHTML`, `document.write`, or jQuery bypasses Angular entirely and is unsafe. Manipulate the DOM through Angular bindings or `Renderer2` (which is also SSR-safe), never by concatenating untrusted strings into markup.

## Trusted Types and CSP

* A **Content Security Policy (CSP)** header restricts which scripts/styles/resources can load, providing defense-in-depth against injection. Angular supports CSP; using a nonce for inline styles is supported via `ngCspNonce`.
* **Trusted Types** (a browser API) locks down dangerous DOM sinks so only sanitized values can reach them; Angular integrates with Trusted Types to further harden against DOM XSS.

## Cross-Site Request Forgery (XSRF/CSRF)

CSRF tricks an authenticated user's browser into making unwanted requests. For cookie-based auth, Angular's `HttpClient` has built-in protection: it reads a token from a cookie (default `XSRF-TOKEN`) and sends it as a header (`X-XSRF-TOKEN`) on mutating requests, which the server validates. Configure via `withXsrfConfiguration`. This requires the server to set and check the token.

## Safe URLs and Navigation

Angular sanitizes `href`/`src` bindings, neutralizing `javascript:` URLs. Still validate user-provided URLs before binding, and be careful with `[href]`/`[src]`/`window.open` on untrusted input. For resource URLs (iframes, scripts), Angular is strict and requires explicit trust — reflecting how dangerous they are.

## Authentication and Token Storage

Angular is a client framework, so tokens live in the browser:

* **Storage** — `localStorage`/`sessionStorage` are readable by any script (XSS-exposed); **HttpOnly cookies** are safer for session tokens since JS can't read them (paired with XSRF protection). Choose based on threat model.
* **Route guards** protect UI routes but are **not** security — the server must always authorize every request. Client-side checks are UX, not enforcement.
* Never embed secrets/API keys in the frontend bundle; anything shipped to the browser is public.

## Dependencies and Supply Chain

Most real-world risk comes through dependencies. Run `npm audit`, keep Angular and libraries updated (`ng update`), pin/lock versions, and vet third-party packages. Avoid `eval`, dynamically built templates, and untrusted `bypassSecurityTrust*` usage.

## Interview Q&A

**Q: How does Angular prevent XSS by default?**
A: It treats bound values as untrusted data and contextually escapes/sanitizes interpolation and property bindings, so injected markup renders as text and dangerous HTML is stripped from `[innerHTML]`.

**Q: What does the sanitizer do?**
A: It cleans values per security context (HTML, style, URL, resource URL), removing scripts and dangerous attributes while preserving safe content.

**Q: What are `bypassSecurityTrust*` methods and their risk?**
A: `DomSanitizer` methods that mark a value as trusted and skip sanitization. Passing user-controlled data to them reintroduces XSS — only use for values you fully control, ideally never.

**Q: Is `[innerHTML]` safe?**
A: The binding is sanitized, but injecting HTML built from user input via native `innerHTML`/`document.write` bypasses Angular and is unsafe. Use bindings or `Renderer2`.

**Q: How does Angular help with CSRF?**
A: `HttpClient` reads an XSRF cookie and sends it as a header on mutating requests (configurable via `withXsrfConfiguration`); the server must issue and validate the token.

**Q: What are CSP and Trusted Types?**
A: CSP is a response header restricting allowed script/resource sources (defense-in-depth); Trusted Types is a browser API locking dangerous DOM sinks to sanitized values. Angular integrates with both.

**Q: Where should you store auth tokens?**
A: HttpOnly cookies (not readable by JS, with XSRF protection) are safer than `localStorage`, which is exposed to any XSS. Choose per threat model and never store secrets in the bundle.

**Q: Do route guards provide security?**
A: No — they're UX gating. The server must authorize every request; client checks can be bypassed.

**Q: How do you handle untrusted URLs?**
A: Rely on Angular's URL sanitization (which blocks `javascript:`), validate user-provided URLs, and treat resource URLs (iframes/scripts) as high-risk requiring explicit trust.

**Q: How do you reduce supply-chain risk?**
A: Keep dependencies updated (`ng update`), run `npm audit`, lock versions, vet packages, and avoid `eval`/dynamic templates/unsafe bypasses.

## Interview Notes

* Explain Angular's secure-by-default XSS model: contextual escaping/sanitization of bindings.
* Warn about `bypassSecurityTrust*` and native `innerHTML` with user input.
* Know CSP and Trusted Types as defense-in-depth.
* Describe built-in XSRF cookie/header protection and server responsibility.
* Discuss token storage trade-offs (HttpOnly cookies vs `localStorage`) and that guards aren't security.
* Mention dependency/supply-chain hygiene and never shipping secrets to the client.
