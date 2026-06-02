# Measuring Web Vitals — LCP & CLS (the way Shopify actually grades you)

Covers BFS 2.1.1 (LCP), 2.1.2 (CLS), 2.1.3 (INP). **Verified against shopify.dev** (Admin installation/OAuth performance + Web Vitals API docs). Read this before trying to "fix performance" — you can waste days optimizing against the wrong tool.

## The one thing to internalize: it's FIELD data, not lab

Shopify measures your admin app's Web Vitals from **real merchant sessions** (RUM), gathered automatically every time a merchant launches your app **through any route**. It is **not** a Lighthouse score.

- **Thresholds (all at the 75th percentile over a rolling 28 days):**
  - **LCP ≤ 2.5s**
  - **CLS ≤ 0.1**
  - **INP ≤ 200ms** (only collected on the **latest App Bridge**)
- **Assessment floor:** your app needs **≥ 100 measurements** for a metric over the last 28 days before that metric is assessed at all. Low-traffic apps simply aren't scored yet → **you must ship and accrue real traffic.** You cannot pass BFS performance from localhost.

> ⚠️ **Lighthouse is unreliable for the admin app.** Shopify's docs say it directly: *"Apps rendered in the Shopify admin run in iFrames, so tools like Lighthouse might not accurately capture performance."* Shopify records Web Vitals in a **separate runtime** from your app. So a green local Lighthouse means little; a red one might be a false alarm. Use the tools below instead.
>
> Lighthouse **is** the right tool for the **storefront** criterion 2.2.1 (your app must not drop the store's Lighthouse score by >10 points) — different surface, different tool.

## Prerequisite: latest App Bridge (3.1.1)

Web Vitals collection requires the **latest App Bridge script in your `<head>`** (it can coexist with a packaged App Bridge dependency — no full migration needed). Without it Shopify can't gather metrics and you can't reach BFS.

```html
<head>
  <meta name="shopify-api-key" content="%SHOPIFY_API_KEY%" />
  <script src="https://cdn.shopify.com/shopifycloud/app-bridge.js"></script>
</head>
```

## Tool 1 — `shopify-debug` overlay (your day-to-day LCP/CLS debugger)

Add the meta flag to your `<head>` and open the console:

```html
<meta name="shopify-debug" content="web-vitals" />
```

You get two log streams:
- **Real-time logs** straight from Web Vitals — including **attribution data for the slow-loading element or inefficient routine**. This is how you find out *which* DOM node is your LCP element and *what* shifted for CLS.
- **Send-time logs** — the final values Shopify actually records (what feeds the Partner Dashboard).

This is the fastest loop: change code → reload the app *inside the admin* → read the attributed LCP/CLS in the console.

## Tool 2 — Web Vitals API (`shopify.webVitals.onReport`) for continuous monitoring

Register a callback to receive every metric and ship it to your own analytics (PostHog, a `/metrics` endpoint, etc.). Lets you track P75 yourself over time and segment by country/route — instead of waiting on the 28-day dashboard.

```html
<head>
  <script src="https://cdn.shopify.com/shopifycloud/app-bridge.js"></script>
  <script>
    function processWebVitals(report) {
      // report.metrics: [{ name: "LCP"|"CLS"|"INP"|"FCP"|"TTFB"|"FID", value, id }, ...]
      // also: report.appLoadId, report.shopId, report.userId, and per-metric country
      navigator.sendBeacon("/metrics/web-vitals", JSON.stringify(report));
    }
    shopify.webVitals.onReport(processWebVitals);   // pass null to unregister
  </script>
</head>
```
Value units: milliseconds for timing metrics; **CLS is unitless**. Pipe these to a dashboard and compute your own P75 so you see regressions in days, not at the 28-day mark.

## Tool 3 — Partner Dashboard (the score of record)

The Partner Dashboard shows the LCP/CLS/INP your app is actually being graded on (P75 / 28d / ≥100 calls). **Discrepancies are normal** between this and other tools: external tools may measure on *every* in-app navigation, while Shopify measures per app-launch — so don't panic if your own RUM numbers differ slightly from the dashboard.

## So how do I actually fix LCP and CLS? (field-first loop)

1. **Reproduce in the real iframe**, not localhost-bare. Open your app inside the Shopify admin with `shopify-debug` on; read the attributed LCP element and CLS sources from the console.
2. **Fix the cause** (see `performance-and-hydration.md` for the catalogue): for **LCP** — shrink/split the JS bundle, don't gate first paint on an auth/data round-trip, defer heavy charts; for **CLS** — match skeleton heights pixel-for-pixel, reserve `<img>` dimensions, keep late/async-gated banners out of flow layout.
3. **Confirm locally** the attributed element/shift is gone via the overlay.
4. **Ship and watch the field** via `onReport` (days) and the Partner Dashboard (the 28-day P75 of record). Real improvement only shows once enough real sessions accrue.

## LCP vs CLS — where each usually comes from in an embedded app

| Metric | Most common admin-app cause | Primary fix |
|---|---|---|
| **LCP** | Big single JS bundle parsed before first paint; app renders `null` until an auth/data fetch resolves (no shell) | Code-split + `lazy()` routes; paint a skeleton shell immediately; defer heavy below-fold JS |
| **CLS** | Skeleton height ≠ real content; async-gated banner/table injected into flow layout after load; images without dimensions | Match skeleton heights; reserve space (min-height / placeholder rows) for variable content; keep late banners `position:fixed`; set `<img width height>` |

## Quick checklist

```
[ ] Latest App Bridge script in <head> (no collection without it)        2.1.x / 3.1.1
[ ] shopify-debug=web-vitals meta added during dev; read attribution     2.1.1 / 2.1.2
[ ] shopify.webVitals.onReport wired to your analytics for P75 tracking
[ ] Verified inside the admin iframe — NOT relying on bare Lighthouse
[ ] Shipped to real merchants; ≥100 calls/28d accruing before judging
[ ] Storefront (2.2.1) checked with Lighthouse separately (≤10pt drop)
```
