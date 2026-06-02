# Design — Banners, Cards, Mobile (iframe) & Accessibility

Covers BFS 4.1.1 (cards/contrast), 4.1.2 (mobile), 4.2.4 (error red), 4.2.6 (previews), 4.3.3 (no auto-pop), 4.3.4 (don't overwhelm), 4.3.6 (dismissible ads).

## 1. No auto-popping banners/modals (4.3.3)

A global, fixed-position banner that shows on every page load = an auto-pop. One app was rejected for exactly this.

- **Fix:** delete the global banner; use an **inline `<s-banner>` / Polaris `Banner`** placed in the page flow, co-located with the toggle that creates the requirement (just-in-time disclosure).
- **Red is for errors only** (4.2.4 / 4.3.3). Warnings use `tone="warning"`, never red CSS.
- **Don't overwhelm** (4.3.4): limit how many banners show at once; organize forms into sections.

## 2. Dismissible ads, persisted (4.3.6)

Promo/cross-sell content must be dismissible and stay dismissed.

```tsx
const [dismissed, setDismissed] = useState(() =>
  typeof window !== "undefined" && localStorage.getItem("promo_dismissed") === "true");
// ...write back on dismiss, wrapped in try/catch for private-mode storage
```

**Web-component event gotcha:** the `dismiss` event on `<s-banner dismissible>` is a **DOM event, not a React synthetic** — wire it with `addEventListener` on a callback ref, not an `onDismiss` prop:
```tsx
const ref = useCallback((node: HTMLElement | null) => {
  node?.addEventListener("dismiss", handleDismiss);
}, [handleDismiss]);
<s-banner ref={ref as any} dismissible>…</s-banner>
```

## 3. Card every data table (4.1.1 / 4.1.2)

Data must sit in a white, rounded, bordered card resembling Shopify admin — desktop table **and** mobile list. Bare rows on the grey page background fail review. Centralize in your shared table component with a `toolbar` slot:
```css
.datalist-card { background:#fff; border:1px solid #e4e5e7; border-radius:12px; overflow:hidden; }
```
`overflow:hidden` clips rows to the rounded corners — a reason to prefer a styled `<div>` over `<s-box>`, whose shadow-DOM overflow is unpredictable. Wrap the loading **skeleton** in the same card so there's no cardless flash.

## 4. The admin-iframe mobile trap (4.1.2) — `@container`, not `@media`

**Critical, and easy to miss:** inside Shopify's admin iframe the viewport is wider than your content column, so **viewport media queries miss the mobile breakpoint**. Use container queries:
```css
.datalist-container { container-type: inline-size; }
@container (max-width: 500px) { .datalist-toolbar { padding: 12px; } }
```

## 5. No data lost to truncation (4.1.2)

- Shorten labels: `"Maximum items per order"` → `"Maximum items"` (context preserved by the heading/helper).
- Add `title=` + `aria-label` to truncated cells.
- Cap dropdown panels: `max-width: calc(100vw - 32px)`.
- On mobile, make dense rows **tap-to-expand** into a `<dl>` key-value grid (56px tap targets) so nothing is hidden (e.g. a `page_url` column that vanished on mobile).
- Reserve space so a floating support widget (Crisp/Intercom) doesn't cover sticky controls: `body { padding-bottom: 88px }` on small screens.

## 6. Status badge correctness

Derive the badge **tone and text from the same source**. A bug shipped where tone came from `event_type` but text from `verification_status`, so color and label disagreed. Use one resolver:
```ts
getStatusTone(status, eventType) // verified→success, dismissed→info, pending→warning, expired→critical
```

## 7. Visible preview (4.2.6)

Show a live, disabled mock that binds to the merchant's current config (background/font color, font size, border radius as `%`, width clamped to `Math.min(width,100)%`) and updates in real time as they change settings. Pair preview labels/inputs with `htmlFor`/`id`; use the right input `type` (`email`, not `text`).

## 8. Onboarding (4.2.2) — two of the published Top-10 rejection reasons

Shopify's "top 10 reasons apps fail BFS" includes **two** onboarding items, so this is high-yield:

- **Show onboarding immediately after install.** No blank/confusing first screen — the merchant lands on a clear guided start. And make it **easy to find again** later (a persistent "Setup guide" entry), not a one-shot you can never reopen.
- **Onboarding must be dismissible and must NOT block core functionality.** Tooltips, guided tours, and checklists can be skipped; the merchant can reach the actual app without completing them. A forced, unskippable flow fails review.
- On uninstall, reset the completion flag so it reappears on reinstall (see `analytics-embed-graphql.md` §3).

## 9. Accessibility — WCAG 2.1 AA (4.1.1)

- **Contrast ≥ 4.5:1.** Greys that FAILED: `#6d7175`, `#aaa`, `#666` → use `#4a4a4a` / `#505050` / `#595959`.
- `aria-label` on icon-only buttons and shape/size radios; `role="progressbar"` + `aria-valuenow/min/max` on progress bars.
- Collapsible rows: `aria-expanded` + 56px tap targets. **Only use `aria-controls` when the target element always exists** — for conditionally-rendered detail panels, rely on `aria-expanded` alone (a dangling `aria-controls` confuses screen readers).
- Reset `expandedId` on filter/page change so a row isn't pre-expanded on a new page.
- Storefront parity: social icons `<div>` → `<button aria-label="Sign in with Google">`; OTP inputs in `<fieldset><legend>` + per-digit `aria-label`; `role="alert"` on errors, `role="status"` + `aria-live="polite"` on spinners.
