# BFS Requirements — Official Criteria + Implementation Notes

**Authoritative source:** https://shopify.dev/docs/apps/launch/built-for-shopify/requirements.md
Always defer to the live page; criteria change. This file mirrors the criteria with a short "what it took" note from real submissions. When a reviewer cites a number, find it here, then jump to the deep-dive reference file named in the last column.

## 1. Prerequisites (business gates — not code)

| # | Title | Note |
|---|---|---|
| 1.1.1 | Meet App Store requirements | App must already be listed and compliant. |
| 1.1.2 | Good Partner standing | No active infractions. |
| 1.2.1 | Minimum installs | **50+ net installs from paid-plan shops.** Ship early to accrue. |
| 1.2.2 | Minimum reviews | **5+ reviews.** Add a review-prompt flow. |
| 1.2.3 | Minimum rating | Meet the recent-rating threshold. |

These block submission **regardless of technical readiness**. Track them separately.

## 2. Performance — Core Web Vitals (mandatory gate) → `measuring-web-vitals.md`, `diagnosing-lcp-cls-from-code.md`, `performance-and-hydration.md`

**This is a hard gate, not a polish item.** Each metric is judged at the **75th percentile over a rolling 28 days**, and needs **≥100 measurements/28d** to even be assessed. Data is **field/RUM**, collected automatically by Shopify on every app launch **via the latest App Bridge** — no App Bridge web-vitals script ⇒ no data ⇒ cannot reach BFS. **Lighthouse is unreliable for the admin app** (it runs in an iframe; Shopify records in a separate runtime).

| # | Title | Threshold | How measured |
|---|---|---|---|
| 2.1.1 | Minimize LCP | ≤ 2.5s | field, App Bridge web vitals |
| 2.1.2 | Minimize CLS | ≤ 0.1 | field, App Bridge web vitals |
| 2.1.3 | Minimize INP | ≤ 200ms | field, **latest App Bridge only** |
| 2.2.1 | Store speed impact | Storefront Lighthouse not reduced > 10 pts | **Lighthouse** (storefront, not admin) |
| 2.3.1 | Checkout speed | p95 ≤ 500ms, ≤ 0.1% failure (≥1,000 req/28d) | server timing |

Tools: `shopify.webVitals.onReport()` (App Bridge) → your analytics; `<meta name="shopify-debug" content="web-vitals">` console overlay (with slow-element attribution); Partner Dashboard for the score of record. To find offenders *before* shipping, audit the code with `diagnosing-lcp-cls-from-code.md`. Biggest wins: skeletons that match content height + reserved space for async/variable content (CLS); bundle code-splitting + no blank-until-data root gate + trimmed loaders (LCP); killed hydration mismatches (LCP, SSR only).

## 3. Integration

| # | Title | Note |
|---|---|---|
| 3.1.1 | Embed in admin | **Latest App Bridge (v4)** loaded from `cdn.shopify.com/shopifycloud/app-bridge.js`. |
| 3.1.2 | Primary workflows in admin | Don't bounce merchants to an external dashboard. |
| 3.1.3 | Seamless sign-up via Shopify creds | No second login. |
| 3.1.4 | Simplified monitoring/reporting | Key metrics on the app home. |
| 3.1.5 | 3rd-party settings in admin | Manage integrations in-app. |
| 3.2.1 | Clean uninstall | Use **theme app extensions** (auto-removed on uninstall). ScriptTags are NOT auto-removed → don't use them. |
| 3.2.2 | No Asset API file writes | Don't create/modify/delete theme files. **Also: ScriptTags are deprecated** — remove `write_script_tags` + all `scriptTagCreate`, and audit `access_scopes` in every `shopify.app*.toml` for `write_themes`/`read_themes`/`write_script_tags` (reviewers flag the scope itself). → `analytics-embed-graphql.md` |

## 4. Design → `polaris-admin-pages.md` + `design-banners-mobile-a11y.md`

### 4.1 Familiar
| # | Title | Note | Ref |
|---|---|---|---|
| 4.1.1 | Follow UX best practices | No custom button colors; content in cards; WCAG AA contrast. | polaris-admin-pages |
| 4.1.2 | Mobile-friendly | Works inside the admin iframe; no horizontal scroll; no truncation data loss. | design-banners-mobile-a11y |
| 4.1.3 | Concise app name | Name must not truncate in nav. | — |
| 4.1.4 | Use the nav menu | `<s-app-nav>` / App Bridge `NavMenu`. | polaris-admin-pages |
| 4.1.5 | **Contextual save bar** | Form changes saved via the contextual save bar, not an inline Save. | polaris-admin-pages |
| 4.1.6 | Use modals appropriately | `<s-modal>` / Polaris `Modal` with heading + action slots. | — |

### 4.2 Helpful
| # | Title | Note | Ref |
|---|---|---|---|
| 4.2.1 | Spelling/grammar | Proofread all copy. | — |
| 4.2.2 | Helpful onboarding | Reset `onboarding_completed` on uninstall so it reappears on reinstall. | — |
| 4.2.3 | Helpful homepage | Setup status + performance metrics on home. | analytics-embed-graphql |
| 4.2.4 | Helpful error messages | Errors in red, next to the relevant field. | design-banners-mobile-a11y |
| 4.2.5 | Guide to logical actions | Most logical action is visually dominant (one primary CTA). | polaris-admin-pages |
| 4.2.6 | Visible previews | Real-time preview of visual customizations. | design-banners-mobile-a11y |

### 4.3 User-friendly
| # | Title | Note | Ref |
|---|---|---|---|
| 4.3.1 | No false claims | Don't guarantee merchant outcomes. | — |
| 4.3.2 | Don't pressure | No countdown timers / guilt copy. | — |
| 4.3.3 | Don't distract | **No auto-popping modals/banners**; minimal animation. | design-banners-mobile-a11y |
| 4.3.4 | Don't overwhelm | Organized forms; limit banners. | design-banners-mobile-a11y |
| 4.3.5 | Don't impersonate Shopify | Distinct branding. | — |
| 4.3.6 | Dismissible ads | All promo content dismissible + persisted. | design-banners-mobile-a11y |
| 4.3.7 | Label & disable premium features | Plan-gated features clearly indicated/disabled. | — |

## 5. Category-Specific (web pixels etc.) → `analytics-embed-graphql.md`

Apps in **Ads (5.1.1), Affiliate (5.2.1), Analytics (5.3.1), Email Marketing (5.6.1), SMS (5.13.1)** must implement **Web Pixel extensions**. Several categories also require **Shopify segments** (5.1.2, 5.6.3, 5.7.1, 5.13.3) and the **visitors API** (5.6.4, 5.7.2, 5.13.4). Other category rules: Carrier Services latency/reliability (5.4.x), Discounts primitives (5.5.x), Fulfillment SLAs (5.8.x), Subscriptions APIs/UX (5.14.x), Reviews flow triggers + block extensions (5.11.x), Returns sync (5.12.x), Invoices print action (5.9.1), Bundles primitives (5.10.1). Check the live page for your category's exact list.

## Cross-cutting (not numbered but enforced)

- **GraphQL-only Admin API** for new public apps (April 2025). Document any justified REST exception (e.g. customer-create needs a password field GraphQL lacks; theme-asset read has no GraphQL equivalent). → `analytics-embed-graphql.md`

## ⚠️ Three-strike rule

Failing the **same** criterion three times = multi-month suspension. Verify a fix against the relevant reference file before resubmitting.
