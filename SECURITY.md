# Security Audit — OdontoMed Planaltina

Scope reviewed: the entire repository at the time of audit
(`index.html`, `unnamed.png`). The project is a single static HTML
landing page served as-is, with no backend, database, server-side code,
or build step.

Because of this, most of the checklist items the owner requested do
not have any attack surface in this repo. The findings below describe
what was checked, what was found, and which items were fixed in the
accompanying PR.

## Summary

| # | Category | Result |
|---|-------------------------------------|--------|
| 1 | Hardcoded API keys / secrets        | None found |
| 2 | SQL injection                       | Not applicable (no backend, no SQL) |
| 3 | Unvalidated user input              | Not applicable (no forms / no JS handling of user input) |
| 4 | Insecure / unpinned dependencies    | **Fixed** — added Subresource Integrity (SRI) to CDN assets |
| 5 | Overly permissive CORS              | Not applicable (no backend) |
| 6 | Exposed debug endpoints             | Not applicable (no backend) |
| 7 | Missing authentication checks       | Not applicable (no authenticated routes) |
| 8 | Reverse tabnabbing (`target="_blank"`) | **Fixed** — added `rel="noopener noreferrer"` on all external links |

## Detailed findings

### 1. Hardcoded API keys and secrets — **None**
No API keys, tokens, database URLs, JWT secrets, service account
credentials, or similar values are present in the repo. The only
identifiers in the source are intentionally public business data:

- Brazilian WhatsApp / phone numbers (`+55 61 99236-4866`, `+55 61 3721-0598`)
- CNPJ `35.692.145/0001-85` (public Brazilian business registration, similar to a US EIN but publicly disclosed on invoices)
- Social media handles (`@odontomedplango`, `OdontoMedPlanG0`)

These are business contact info and are meant to be public. No action
required.

### 2. SQL injection — **Not applicable**
There is no backend code, ORM, or SQL in the repository.

### 3. Unvalidated user input — **Not applicable**
The page has no `<form>`, no `<input>`, no `fetch`/`XHR` calls, and
does not read from `location.hash`, `location.search`,
`postMessage`, or any other user-controllable source. The inline JS
only:

- hides a loading screen,
- toggles mobile-nav visibility on a static button,
- scrolls to same-page anchors,
- initializes AOS animations.

No DOM sinks (`innerHTML`, `document.write`, `eval`, etc.) are used in
first-party code, so there is no XSS vector in application code.

Note: `cdn.tailwindcss.com` itself uses `Function(...)`/`eval` at
runtime. That is Tailwind's documented behavior for the Play CDN and
is the reason it is not recommended for production (see item 4).

### 4. Dependencies / supply chain — **Fixed**
All third-party assets are loaded from public CDNs:

| Asset | URL |
|-------|-----|
| Tailwind Play CDN | `https://cdn.tailwindcss.com` |
| Font Awesome 6.4.0 | `https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css` |
| Google Fonts (Poppins) | `https://fonts.googleapis.com/css2?...` |
| AOS 2.3.1 CSS | `https://unpkg.com/aos@2.3.1/dist/aos.css` |
| AOS 2.3.1 JS | `https://unpkg.com/aos@2.3.1/dist/aos.js` |

Risks before the fix:

- **No Subresource Integrity (SRI).** If any of these CDNs (or a
  mutable unpkg/tailwind endpoint) were compromised or the file were
  swapped out, the page would execute arbitrary attacker-controlled
  JavaScript with full access to the page — effectively a stored XSS
  against every visitor.
- The Tailwind **Play CDN is explicitly "not for production"** per
  Tailwind's own docs. It ships the full JIT compiler, runs `eval`
  in the browser, and serves dynamic responses that cannot be pinned
  with SRI.

Fixes applied in this PR:

- Added `integrity="sha384-…"` + `crossorigin="anonymous"` +
  `referrerpolicy="no-referrer"` to Font Awesome, AOS CSS, and AOS JS
  (these are the three external assets served as immutable versioned
  files, so SRI works cleanly).
- Left Google Fonts without SRI because the `css2?...` endpoint
  returns a different stylesheet depending on the user-agent
  (woff/woff2 variants), so a fixed hash would break the site for
  some browsers.
- Left the Tailwind Play CDN `<script>` without SRI for the same
  reason — the endpoint is dynamic. See "Recommended follow-ups" for
  how to remove this runtime dependency entirely.

### 5. CORS — **Not applicable**
There is no server, no `fetch` calls to APIs, and no
`Access-Control-Allow-*` headers in the repository. Whatever static
host serves `index.html` controls the response headers.

### 6. Exposed debug endpoints — **None**
No admin, debug, test, or internal routes exist, because there is no
server in this repo.

### 7. Missing authentication checks — **Not applicable**
There is no authenticated functionality in the site. Every link is a
public anchor or a public `https://wa.me/...` / `tel:` deep-link.

### 8. Reverse tabnabbing (`target="_blank"` without `rel`) — **Fixed**
All nine outbound `target="_blank"` links (WhatsApp, Instagram,
Facebook, developer contact, floating WhatsApp button) previously
lacked `rel="noopener noreferrer"`. On older browsers and in some
embedded web views, a `target="_blank"` link gives the destination
page access to `window.opener`, which it can use to silently
redirect the original tab to a phishing page ("reverse tabnabbing").

Fix applied: every `target="_blank"` link now carries
`rel="noopener noreferrer"`. `noopener` severs the `window.opener`
reference; `noreferrer` additionally prevents the destination from
seeing the `Referer` header.

## Recommended follow-ups (not fixed in this PR)

These are lower-priority hardening items. They were not applied
automatically because each requires a small product decision by the
owner.

1. **Drop the Tailwind Play CDN in favor of a compiled CSS file.**
   Tailwind's own documentation states the Play CDN is not for
   production: it is heavier, runs `eval` in the browser, and cannot
   be integrity-checked. Running `npx tailwindcss -o styles.css
   --minify` once and committing `styles.css` removes a large chunk
   of supply-chain risk and also removes the need for `unsafe-eval`
   if a CSP is added (item 3 below).

2. **Self-host Font Awesome, AOS, and Poppins.** For a site this
   small, hosting the few KB of CSS/JS/fonts alongside `index.html`
   eliminates CDN supply-chain risk entirely, removes the need for
   SRI, and also improves privacy (no visitor data leaking to
   unpkg/cdnjs/Google).

3. **Add a Content Security Policy.** Once the Tailwind Play CDN is
   removed, a strict CSP becomes practical, e.g.:

   ```html
   <meta http-equiv="Content-Security-Policy"
         content="default-src 'self';
                  img-src 'self' https://images.unsplash.com data:;
                  style-src 'self' 'unsafe-inline';
                  script-src 'self';
                  frame-src https://www.google.com;
                  connect-src 'self';
                  object-src 'none';
                  base-uri 'self';
                  form-action 'self';
                  frame-ancestors 'none';">
   ```

   A CSP was intentionally not added in this PR because the current
   Tailwind Play CDN usage requires `'unsafe-eval'` in `script-src`,
   which defeats most of the point of having a CSP.

4. **Serve security headers at the hosting layer.** Whichever host
   serves this site (Netlify, Vercel, Nginx, Apache, GitHub Pages
   behind a proxy, etc.) should set at minimum:

   - `Strict-Transport-Security: max-age=31536000; includeSubDomains`
   - `X-Content-Type-Options: nosniff`
   - `Referrer-Policy: strict-origin-when-cross-origin`
   - `Permissions-Policy: geolocation=(), microphone=(), camera=()`

   These live in the hosting config, not in the HTML file, so they
   are out of scope for this repo but worth mentioning.

5. **Pin the embedded Google Maps `src`.** The current iframe URL
   contains placeholder-looking coordinates
   (`!3d-15.4567...!2d-47.6123...`). That is not a security bug —
   it just points to the wrong location. Replace it with the real
   `Embed a map` URL from Google Maps for `QC 03 MC Lote 1C,
   Planaltina-GO`.

## What was fixed in this PR

- Added SRI + `crossorigin` + `referrerpolicy` to Font Awesome, AOS
  CSS, and AOS JS (`<ref_snippet file="/home/ubuntu/repos/Odontomed-/index.html" lines="12-27" />` and `<ref_snippet file="/home/ubuntu/repos/Odontomed-/index.html" lines="703-707" />`).
- Added `rel="noopener noreferrer"` to every `target="_blank"` link
  in the page (9 occurrences).
