# Diagnosing LCP & CLS From Code (static triage)

Performance is a **mandatory BFS gate**, not a nice-to-have: LCP ≤2.5s and CLS ≤0.1 at P75/28d are required, and an app that doesn't load the App Bridge Web Vitals script produces *no* data and can't be assessed at all. Field data tells you *that* you're failing; this file tells you *why*, **from the code**, before you ship — so you can predict and locate the offenders during a review.

How to use: walk LCP (4 sub-parts) then CLS (6 causes). Each lists the **detectable code signal** to grep/read for in an embedded Shopify app, and the fix. Then run the **grep toolkit** and produce a **ranked risk list**. Confirm hits against the real field tooling in `measuring-web-vitals.md`.

---

## LCP — break it into 4 sub-parts (web.dev)

LCP = TTFB + Resource Load Delay + Resource Load Duration + Element Render Delay. Target split: TTFB + load-duration ≈ 80%, the two delays <10% each. Optimizing one part often just shifts time to another — find the dominant one via `shopify-debug` attribution, then match it to code here.

### 1. TTFB — server/first-byte (~40%)
**Code signals**
- Auth/redirect chains before HTML: `afterAuth` doing heavy work; multiple `redirect()` hops in the OAuth path; loaders awaiting slow upstreams **serially**.
- A loader/controller that fetches everything before returning (no `Promise.all`, no streaming).
- Querying far more than you render (e.g. a 44-field shop query when you need 4).

**Fix:** parallelize independent loader calls (`Promise.all`); query only needed fields; keep `afterAuth` <50–100ms (defer the rest); stream SSR (`renderToPipeableStream`). Don't block first byte on third-party calls.

### 2. Resource Load Delay — gap before the LCP resource starts (<10%)
**Code signals**
- LCP element (hero image/heading) **added by JS**, not in initial HTML/SSR output.
- `loading="lazy"` on an above-the-fold / LCP image.
- LCP image referenced only from CSS `background-image` or a JS import (preload scanner can't find it).
- Critical asset on a different origin with no `<link rel="preconnect">`.

**Fix:** put the LCP element in the initial HTML; never `loading="lazy"` above the fold; `<link rel="preload">` the LCP image and `preconnect` its origin; reference it as a real `<img src>`.

### 3. Resource Load Duration — transferring the bytes (~40%)
**Code signals**
- Large unoptimized images (no `.webp`/`.avif`, no responsive `srcset`); oversized fonts.
- `font-display: auto`/`block` (render-blocking font).
- **Big JS bundle** parsed/loaded before paint — in an admin SPA this is usually the LCP bottleneck. `webpack.config.js` with no `splitChunks`; a single multi-hundred-KB chunk; `mode` not `'production'` (no minification); zero dynamic `import()` / `React.lazy`.
- Missing `cache-control` on static assets.

**Fix:** compress/serve modern image formats; `font-display: optional` + preload; **code-split** (`optimization.splitChunks: { chunks: 'all' }` in webpack, route-level `React.lazy()`/`Suspense`); ensure prod minification; cache static assets.

### 4. Element Render Delay — bytes done but not painted yet (<10%)
**Code signals (the big CSR trap)**
- **App renders `null`/blank until data resolves:** `{loading ? null : children}`, `{!data ? <Spinner/> : <Page/>}` at the app root — first paint waits on a network round-trip. **This is the #1 SPA LCP killer.**
- Render-blocking `<script>` in `<head>` without `async`/`defer`; large render-blocking CSS.
- Client-side rendering where the LCP element only exists after JS executes (no SSR/skeleton).
- Long main-thread tasks (heavy synchronous work on mount).
- A/B-test or feature-flag libraries that hide content until they decide.

**Fix:** **paint a shell/skeleton immediately**, gate only the data-dependent inner content; `defer` non-critical scripts; SSR or pre-render the LCP element; move heavy work off the mount path; don't hide the page waiting on flags.

---

## CLS — 6 causes (web.dev), each with a code signal

CLS = unexpected movement of already-rendered content. Shifts within 500ms of a user interaction don't count — everything else does.

| # | Cause | Detectable code signal | Fix |
|---|---|---|---|
| 1 | **Images without dimensions** | `<img src…>` with no `width`/`height` and no CSS `aspect-ratio` | Set `width`+`height` (browser derives aspect-ratio) |
| 2 | **Embeds/iframes/slots without reserved space** | empty `<div id="…">` filled later; charts/maps/`<iframe>` with no sized container | Reserve `min-height` or `aspect-ratio` on the container |
| 3 | **Dynamically injected content above existing content** | `insertAdjacentHTML('afterbegin', …)`; a banner/notice rendered **after an async fetch** into normal flow at the top of the page; `null`-until-loaded banner that then takes flow space | Reserve a fixed-height placeholder; or render the banner `position:fixed`/sticky; or decide visibility from a *synchronous* source before paint |
| 4 | **Web fonts (FOIT/FOUT)** | `@font-face` with default `font-display`; `font-family` with no/poor fallback | `font-display: optional`; matching fallback; `preload` critical fonts |
| 5 | **Animations that trigger layout** | `@keyframes`/transitions animating `top`/`left`/`width`/`height`/`margin` | Animate `transform`/`opacity` only |
| 6 | **Network response then DOM update (no reserved space)** | `fetch().then(… setState(list))` where the list/table grows a container **from zero**; variable-height content with no skeleton | Reserve space (min-height or N placeholder rows at real row height) before data arrives |

### Embedded-app specials (seen in real audits)
- **Skeleton height ≠ real content height.** A `h-[300px]` skeleton under `h-[500px]` content = guaranteed shift. Skeletons must match **pixel-for-pixel**, including every sub-element.
- **`ClientOnly`/SSR-gated banner dropped into flow layout** after the client query resolves → pushes `<Outlet/>`/page down. Keep it `position:fixed`. (SPA equivalent: *any* element whose render is gated on an async fetch.)
- **Polling/late-mounting banners** (`setInterval(fetchStatus, …)`) that appear/disappear in flow are recurring CLS, not a one-time hit.
- **Conditional whole-page branches** keyed on a late-read value (e.g. `usesExtension` meta read after first paint) that swap a short page for a tall one.

---

## Grep toolkit (run these at the repo root)

```bash
# LCP — bundle / code-splitting / blank-until-data
grep -rn "splitChunks\|optimization" webpack.config.js vite.config.* 2>/dev/null   # any code-splitting?
grep -rn "React.lazy\|lazy(\|import(" resources src app 2>/dev/null                 # dynamic imports present?
grep -rn "loading ? null\|!data\|isLoading ? null\|return null" --include=*.js --include=*.tsx  # blank-until-data root gate
grep -rn 'loading="lazy"' --include=*.js --include=*.tsx                            # lazy on (maybe) LCP image
grep -rn "<script" public index.html app/root.* | grep -v "async\|defer\|module"   # render-blocking scripts in head

# CLS — images, injected content, fonts, layout animations
grep -rn "<img" --include=*.js --include=*.tsx | grep -v "width=\|height="          # imgs missing dimensions
grep -rn "insertAdjacentHTML\|document.body.appendChild\|createElement(\"style\")"  # JS-injected DOM/chrome
grep -rn "setInterval\|fetchStatus\|poll" --include=*.js | grep -i "banner\|status" # polling banners
grep -rn "font-display" --include=*.css                                            # should be 'optional'/'swap'
grep -rn "@keyframes" -A4 --include=*.css | grep -E "top:|left:|width:|height:|margin"  # layout-animating keyframes
grep -rn "skeleton\|Skeleton" --include=*.js --include=*.tsx                        # do skeletons exist at all?

# App Bridge web-vitals prerequisite — if MISSING, no data => cannot be assessed
grep -rn "app-bridge.js\|shopify.webVitals\|shopify-debug" public app index.html *.erb
```

## Framework predictors (where the risk concentrates)

- **CSR SPA (webpack/Vite, React 17/18, no SSR):** dominant risks are **LCP** — one big bundle (no `splitChunks`/`lazy`) and **blank-until-auth/data** root gating; **CLS** — variable-height lists/tables and late banners with no reserved space. Hydration section does **not** apply (no SSR).
- **SSR (Remix / React Router 7):** dominant risks are **hydration mismatches** (locale formatting, `Date.now()` at render, SSR↔client flags) which discard SSR HTML and tank LCP; serial loaders (TTFB); skeleton mismatch (CLS). See `performance-and-hydration.md`.
- **Polaris (React or Web Components):** Polaris components are generally CLS-safe, but **your** skeletons, images, custom CSS, and async-gated banners are not. `<s-table>` self-cards (don't over-flag), but fixed column widths can defeat its responsive mode.

## Triage output (what to produce per repo)

For each hit, record: **metric (LCP/CLS) · file:line · sub-part/cause # · severity · fix**. Rank by likely field impact:
1. **App Bridge web-vitals script missing** → blocker (no assessment possible).
2. **Blank-until-data root gate** / **single un-split bundle** → high LCP.
3. **Async-gated banner in flow layout** / **skeleton-height mismatch** / **variable list from zero** → high CLS.
4. Images without dimensions, fonts without `font-display`, layout-animating keyframes → medium CLS.
5. Render-blocking scripts, missing preconnect/preload → medium LCP.

Then verify the top items in the real admin iframe with `shopify-debug=web-vitals` attribution (`measuring-web-vitals.md`) — field data is the arbiter; static triage just tells you where to look.
