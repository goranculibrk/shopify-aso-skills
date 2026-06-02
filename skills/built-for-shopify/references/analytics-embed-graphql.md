# Web Pixel, App-Embed Status & GraphQL-only

Covers BFS 5.x.1 (web pixels), 4.2.3 (homepage setup status), and the cross-cutting GraphQL-only Admin API rule.

## 1. Web pixel extension (5.1.1 / 5.3.1 / 5.6.1 / 5.13.1)

Ads, Analytics, Affiliate, Email-marketing, and SMS apps must subscribe to standard customer events via a Web Pixel extension.

`extensions/<app>-pixel/shopify.extension.toml`:
```toml
type = "web_pixel_extension"
runtime_context = "strict"
[customer_privacy]
analytics = true
marketing = false
preferences = false
sale_of_data = "disabled"
[settings]
type = "object"
```
`src/index.ts`:
```ts
import { register } from "@shopify/web-pixels-extension";
register(({ analytics }) => {
  analytics.subscribe("page_viewed", () => {});   // the standard event BFS measures
  analytics.subscribe("app:custom_event", () => {});
});
```
Activate **idempotently** in `afterAuth` (fire-and-forget; ignore "already exists"):
```ts
const data = await admin.graphql(
  `mutation { webPixelCreate(webPixel:{settings:"{}"}){ userErrors{message} webPixel{id} } }`);
const errs = data?.webPixelCreate?.userErrors;
if (errs?.length && !errs.some(e => e.message.toLowerCase().includes("already"))) logger.warn(errs);
```
Publish custom events from the storefront **best-effort, never blocking**:
```ts
if (window.Shopify?.analytics?.publish)
  window.Shopify.analytics.publish("app:custom_event", { /* ...payload... */ });
```

## 2. Permanent app-embed status indicator (4.2.3)

The merchant must always see whether your theme app embed is enabled, on the homepage.

- **Permanent pill** in the Polaris `Page` `titleMetadata` on every page, via a wrapper component that injects it. Web Components: `<s-badge tone={enabled ? "success" : "warning"}>` on the homepage.
  ```tsx
  // Polaris React wrapper that injects the pill into every page
  export function AppPage(props) {
    const titleMetadata = props.titleMetadata ?? <AppEmbedStatusPill />;
    return <Page {...props} titleMetadata={titleMetadata} />;
  }
  ```
- **Single source of truth:** the pill and any CTA banner read the **same** React Query key (`useEmbedDetection`) so they never disagree. Return `null` while loading to avoid a disabled→enabled flash.
- **Theme-editor deep link** that auto-activates the embed:
  ```ts
  `https://${shop}/admin/themes/current/editor?context=apps&template=index&activateAppId=${uuid}/${EXTENSION_NAME}`
  ```
- Refetch embed status on `visibilitychange` (tab focus) and optionally poll for a few seconds after the merchant returns from the theme editor; stop polling once it flips enabled.

Only make the **warning banner dismissible** *because* the permanent pill already communicates status (4.3.6 vs "always show status").

## 3. Clean uninstall, no Asset API, no ScriptTags (3.2.1 / 3.2.2)

- Ship the storefront surface as a **theme app extension** (app embed / app block) — Shopify removes it automatically on uninstall. Don't write theme files via the Asset API.
- **ScriptTags are deprecated and a BFS fail.** They are *not* auto-removed on uninstall (so they break 3.2.1) and the deprecated pattern itself gets flagged. Remove every `scriptTagCreate`/`scriptTagDelete` call and stop injecting on install/settings-save; move all storefront code into the theme app extension.
- **Audit `access_scopes` as a first-pass signal.** Reviewers flag the *scopes* even without a live call. Grep every config — scopes drift across variants:
  ```bash
  grep -n "scopes" shopify.app*.toml
  # 🚩 write_script_tags, write_themes, read_themes  → remove if not strictly justified
  ```
  `write_themes`/`read_themes` = Asset API capability (3.2.2); `write_script_tags` = deprecated ScriptTags. Removing them is usually part of the theme-extension migration.
- **Verify the migration is actually wired.** A half-built "switch to our new extension" banner that calls endpoints which 404 (and polls them) is worse than nothing — it's a broken auto-popping banner. Confirm the status/activate endpoints exist and the embed-detection source is real, not a dead route. (Real audit finding.)
- On the uninstall webhook, reset `onboarding_completed` so onboarding reappears on reinstall (4.2.2).

## 4. GraphQL-only Admin API (April 2025, cross-cutting)

New public apps must use GraphQL for the Admin API.

- Migrate customer search/read to GraphQL (e.g. a `shopifyGraphql.ts` helper).
- **Document any justified REST exception:**
  - customer-create — GraphQL `customerCreate` lacks a `password` field.
  - theme-asset read — no GraphQL equivalent.
- Drop `restResources` from the Shopify app server config once migrated.
- Query only the fields you need (also a perf win — see `performance-and-hydration.md`).
