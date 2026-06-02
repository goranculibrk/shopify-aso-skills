---
name: built-for-shopify
version: 1.0.0
description: "Use when building, reviewing, or submitting a Shopify embedded app for the Built for Shopify (BFS) badge — Polaris admin pages, Core Web Vitals (LCP/CLS/INP), contextual save bar, web pixels, app-embed status, or fixing a BFS review rejection."
---

# Built for Shopify (BFS) — Embedded App Compliance

A field guide to building Shopify embedded-app admin UIs that pass the **Built for Shopify** review. Distilled from two apps that earned (and re-earned, across rejection rounds) the badge using two different UI stacks:

- **Polaris React** (`@shopify/polaris` v13, Remix)
- **Polaris Web Components** (`<s-*>` from `cdn.shopify.com/shopifycloud/polaris.js`, React Router 7)

The BFS criteria are **stack-agnostic** — this skill maps each requirement to a concrete implementation in both stacks.

**Official source of truth (always defer to it):** https://shopify.dev/docs/apps/launch/built-for-shopify/requirements.md
Read `references/requirements.md` for the full criterion list mirrored here with the per-criterion implementation notes.

## When to use

- Building or refactoring **admin pages** for a Shopify embedded app and you want them BFS-ready.
- A **BFS review was rejected** and you need to map the reviewer's criterion numbers (e.g. 4.1.2, 4.3.3) to fixes.
- Wiring the **contextual save bar**, replacing custom buttons, fixing **CLS/LCP/INP**, adding a **web pixel**, or an **app-embed status indicator**.
- Deciding between **Polaris React vs Polaris Web Components**.

## First decision: which Polaris?

| | Polaris React | Polaris Web Components |
|---|---|---|
| Import | `import { Button } from "@shopify/polaris"` | none — `<s-button>` defined at runtime by `polaris.js` |
| Bundle | ships Polaris + Emotion in your JS | near-zero; loads from Shopify CDN |
| Booleans | normal props | **attributes** — pass `value || undefined` to omit |
| Events | React synthetic props | DOM events; custom ones (`dismiss`) need `addEventListener` on a ref |
| Restyle internals | className + tokens | **shadow DOM blocks your CSS** — use native els + classes |
| Gaps | mature | some components missing (`<s-button-group>` does NOT exist) |

New app → **Web Components** (smaller bundle, future direction, easier CWV). Existing Polaris-React app → stay. App Bridge (save bar, nav, deep links) is identical either way and always loads from the CDN. **Use the latest App Bridge v4** (criterion 3.1.1).

## Criterion → implementation map

| BFS # | Requirement | Where it's covered |
|---|---|---|
| 1.2.1 / 1.2.2 / 1.2.3 | 50+ installs, 5+ reviews, rating threshold | business gate — not code; ship early to accrue |
| 2.1.1 / 2.1.2 / 2.1.3 | LCP ≤2.5s, CLS ≤0.1, INP ≤200ms (P75/28d) | `references/performance-and-hydration.md` |
| 3.1.1 | Embed in admin, latest App Bridge | App Bridge v4 from CDN |
| 3.1.4 | Monitoring/metrics on app home | dashboard homepage |
| 3.2.1 / 3.2.2 | Clean uninstall, no Asset API | theme app extensions, not theme files |
| 4.1.1 | Mimic admin look & feel | `references/polaris-admin-pages.md` (buttons, cards) |
| 4.1.2 | Mobile-friendly | `references/design-banners-mobile-a11y.md` (the iframe trap) |
| 4.1.4 | Use the nav menu | `<s-app-nav>` / App Bridge `NavMenu` |
| 4.1.5 | **Contextual save bar** | `references/polaris-admin-pages.md` |
| 4.1.6 | Use modals appropriately | `<s-modal>` / Polaris `Modal` with heading + action slots |
| 4.2.2 | Helpful onboarding | reset onboarding on reinstall |
| 4.2.3 | Helpful homepage + setup status | `references/analytics-embed-graphql.md` (status pill) |
| 4.2.4 | Errors in red next to fields | only red usage allowed |
| 4.2.5 | One logical/dominant action | one primary CTA per view |
| 4.2.6 | Visible previews | live preview component |
| 4.3.3 | Don't distract (no auto-pop modals) | `references/design-banners-mobile-a11y.md` |
| 4.3.6 | Dismissible ads | dismissible + persisted banners |
| 4.3.7 | Label/disable premium features | plan-gated UI |
| 5.x.1 | Web pixels (ads/analytics/email/SMS) | `references/analytics-embed-graphql.md` |
| — | GraphQL-only Admin API (Apr 2025) | `references/analytics-embed-graphql.md` |

## Reference files (load the one you need)

- **`references/requirements.md`** — every official criterion + the implementation note for each. Start here when triaging a rejection.
- **`references/measuring-web-vitals.md`** — how Shopify *actually* grades LCP/CLS/INP (field/RUM, P75/28d, ≥100 calls), the App Bridge `shopify.webVitals.onReport` API, the `shopify-debug` overlay, and why Lighthouse is unreliable for the admin iframe. **Read before "fixing performance."**
- **`references/diagnosing-lcp-cls-from-code.md`** — predict & locate LCP/CLS offenders *from the code*: the 4 LCP sub-parts and 6 CLS causes mapped to detectable code signals, a grep toolkit, and framework-specific risk predictors. Use this to audit a repo.
- **`references/performance-and-hydration.md`** — the fix catalogue: CLS skeleton matching, LCP loader trimming + bundle/code-splitting, lazy charts, SSR vs client-SPA split, and the full list of SSR hydration mismatch causes + fixes.
- **`references/polaris-admin-pages.md`** — contextual save bar, buttons, cards, the Polaris-React↔Web-Components migration cheat-sheet, Tailwind-as-layout rule.
- **`references/analytics-embed-graphql.md`** — web pixel extension + idempotent `webPixelCreate`, permanent app-embed status indicator + theme-editor deep link, ScriptTag-deprecation / `access_scopes` audit, GraphQL-only migration.
- **`references/design-banners-mobile-a11y.md`** — no auto-pop banners, dismissible/persisted ads, card-wrapping tables, the `@container`-not-`@media` admin-iframe trap, mobile truncation, WCAG 2.1 AA.

## What actually gates you vs. what gets you rejected

Two different walls, both real:

- **Performance is a hard, automated gate.** LCP ≤2.5s and CLS ≤0.1 (P75/28d) are *mandatory*. Worse, they're silent: without the latest App Bridge web-vitals script there's **no data**, and under **100 measurements/28d** a metric **can't be assessed** — so a low-traffic or mis-instrumented app is blocked before a human ever looks. Instrument and measure first (`references/measuring-web-vitals.md`); audit the code for offenders (`references/diagnosing-lcp-cls-from-code.md`).
- **Design/UX is where humans reject you.** Shopify's published "top 10 reasons apps fail BFS review" are **all design criteria (4.x), zero performance** — because perf is pre-filtered by the gate above. The actual rejections, verbatim:

  1. Theme-block status not shown on the homepage (**4.2.3**)
  2. Form lets you navigate away without the contextual save bar (**4.1.5**)
  3. Unsolicited/auto-popping modals or popovers (**4.3.3**)
  4. Red used for anything but errors (**4.3.3**)
  5. Layout breaks on mobile (**4.1.2**)
  6. Custom save button instead of the contextual save bar (**4.1.5**)
  7. Promo content not permanently dismissible (**4.3.6**)
  8. Onboarding not dismissible / blocks core functionality (**4.2.2**)
  9. Features unreachable on mobile (**4.1.2**)
  10. Onboarding not shown immediately after install / hard to find again (**4.2.2**)

  Treat these as the highest-yield checklist — they're what reviewers fail apps for most.

## ⚠️ Three-strike rule

Failing the **same** criterion 3× triggers a multi-month suspension. Do not resubmit a flagged criterion until you're confident it's actually fixed — verify against the reference file, don't guess.

## Build-order checklist (paste into your BFS ticket)

```
[ ] Latest App Bridge v4 from CDN; nav menu (<s-app-nav>/NavMenu)         3.1.1 / 4.1.4
[ ] App Bridge web-vitals script in <head> (no data => can't be assessed) 2.1.x
[ ] Contextual save bar on every settings form (no inline Save)            4.1.5
[ ]   ...and it's the page's PRIMARY action — demote in-page primaries     4.2.5
[ ] All buttons = Polaris variants; no hardcoded colors/emoji              4.1.1
[ ] Red reserved for errors only, next to the field                        4.2.4 / 4.3.3
[ ] One primary CTA per view                                               4.2.5
[ ] Every data table wrapped in a white rounded card (desktop + mobile)    4.1.1 / 4.1.2
[ ] @container queries (container-type: inline-size), NOT @media           4.1.2
[ ] No truncation data loss: short labels + title= on cells                4.1.2
[ ] No auto-popping global banners; inline banners only                    4.3.3
[ ] Promo/ad banners dismissible + persisted (localStorage, try/catch)     4.3.6
[ ] Premium features labelled + disabled when plan-gated                   4.3.7
[ ] Onboarding shown immediately after install, dismissible, non-blocking  4.2.2
[ ] Visible live preview of visual customizations                          4.2.6
[ ] Permanent app-embed status indicator on homepage + theme deep link     4.2.3
[ ] Skeletons match real content height; aria-hidden; same card wrapper    2.1.2
[ ] <img> has width/height; late banners stay position:fixed               2.1.2
[ ] No locale formatting / Date.now() / SSR-vs-client flags in render      2.1.1
[ ] lang="en" on <html>; client-fetch volatile tables; ClientOnly          2.1.1
[ ] Web pixel subscribed to page_viewed; webPixelCreate idempotent         5.x.1
[ ] No deprecated ScriptTags; audit access_scopes (no write_themes/        3.2.2
[ ]   write_script_tags) across all shopify.app*.toml
[ ] GraphQL-only Admin API (document any justified REST exception)         —
[ ] WCAG 2.1 AA: contrast ≥4.5:1, aria-labels, aria-expanded               4.1.1
[ ] Reset onboarding_completed on uninstall webhook                        4.2.2
[ ] Clean uninstall via theme app extension; no Asset API writes           3.2.1 / 3.2.2
[ ] CWV verified in the admin iframe via shopify-debug + onReport,          2.1.x
[ ]   not bare Lighthouse; P75/28d LCP ≤2.5s, CLS ≤0.1, INP ≤200ms
[ ] Business gates: 50+ installs, 5+ reviews, rating threshold             1.2.x
[ ] DON'T resubmit a flagged criterion until truly fixed (3-strike)
```
