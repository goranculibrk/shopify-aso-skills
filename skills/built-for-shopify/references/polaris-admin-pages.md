# Polaris Admin Pages — Save Bar, Buttons, Migration

Covers BFS 4.1.1 (look & feel), 4.1.4 (nav), 4.1.5 (save bar), 4.2.5 (logical action).

## 1. Contextual save bar (4.1.5) — no inline Save button

Settings forms must use the App Bridge contextual save bar that appears on dirty state with a Discard action.

### Polaris React (`@shopify/app-bridge-react`)

```tsx
import { SaveBar } from "@shopify/app-bridge-react";

// Dirty = local state differs from loader data
const isDirty = useMemo(() =>
  appMode !== data.appMode || customMessage !== (data.customMessage ?? ""),
[appMode, customMessage, data]);

const handleDiscard = useCallback(() => {
  setAppMode(data.appMode);
  setCustomMessage(data.customMessage ?? "");   // reset EVERY field
}, [data]);

<SaveBar id="settings-save-bar" open={isDirty} discardConfirmation>
  {/* children are BARE <button>s — App Bridge renders them as the native save/discard */}
  <button
    variant="primary"
    onClick={handleSave}
    loading={isUpdating ? "" : undefined}     // "" / undefined, NOT a boolean
    disabled={isUpdating || hasError}         // guard BOTH buttons vs double-save
  ></button>
  <button onClick={handleDiscard} disabled={isUpdating}></button>
</SaveBar>
```

### Polaris Web Components (App Bridge `<ui-save-bar>` + global `shopify`)

`<ui-save-bar>` is App Bridge (loaded from the CDN), so the dirty-tracking logic is identical to the React version above — only the show/hide call differs.

```tsx
const dataLoaded = useRef(false);                 // guard so initial load doesn't trip the bar
useEffect(() => { dataLoaded.current = true; }, [data]);

// Dirty = local field state differs from loader data (compare EVERY field)
const isDirty = useMemo(() =>
  dataLoaded.current && (
    appMode !== data.appMode || customMessage !== (data.customMessage ?? "")
  ), [appMode, customMessage, data]);

const handleSaveBar = useCallback(async () => { await save({ appMode, customMessage }); }, [appMode, customMessage]);
const handleDiscardBar = useCallback(() => {
  setAppMode(data.appMode);
  setCustomMessage(data.customMessage ?? "");     // reset EVERY field
}, [data]);

useEffect(() => {
  if (isDirty) shopify.saveBar.show("settings-save-bar");
  else shopify.saveBar.hide("settings-save-bar");
}, [isDirty]);

<ui-save-bar id="settings-save-bar">
  <button variant="primary" onClick={handleSaveBar} loading={loader || undefined}></button>
  <button onClick={handleDiscardBar}></button>
</ui-save-bar>
```

**Gotchas**
- Save-bar children are **bare `<button>`** with App Bridge attributes — not Polaris `<Button>`/`<s-button>`.
- `loading` takes `""` / `undefined`, never `true` / `false`.
- Disable **both** buttons while saving or you allow a double-save.
- Track dirty state behind a `dataLoaded` ref so the initial hydration doesn't trip the bar.

## 2. Buttons (4.1.1) — never hardcode colors

Banned: `bg-[#008060]`, `bg-green-*`, per-plan CTA color maps, `tone="success"`/`tone="critical"` as decoration, emoji in labels (🚫/✅). Red/`critical` is reserved for **destructive actions and field errors only** (4.2.4 / 4.3.3).

```tsx
// BEFORE — custom button
<button className="bg-[#008060] hover:bg-[#006e52] text-white px-4 py-1.5 rounded-lg">Enable</button>
// AFTER — Polaris variant (also gives you the loading spinner free)
<Button variant="primary" loading={isSubmitting} onClick={handleActivate}>Enable</Button>
<Button variant="tertiary" onClick={handleDismiss}>Later</Button>
```

Web Components: `<s-button variant="primary">` / `variant="secondary">`. **`<s-button-group>` does NOT exist** — wrap in a flex `<div>`.

One primary CTA per view (4.2.5): use a single "next step" card, not a multi-button stepper. **When a contextual save bar can appear, its Save *is* the view's primary action** — so demote any in-page `variant="primary"` button to `secondary`/`tertiary`, or convert inline enable/disable to `<s-switch>`, so they don't compete with the save bar. (Real audit finding: a settings page had 3 competing primaries — the save bar + an in-page Enable + a "Reinstall App" button.)

## 3. Nav menu (4.1.4)

```tsx
// Web Components
<s-app-nav>
  <s-link href="/app" rel="home">Dashboard</s-link>
  <s-link href="/app/settings">Settings</s-link>
</s-app-nav>
// Polaris React: <NavMenu> from @shopify/app-bridge-react
```
(`rel="home"` is valid per Shopify docs but may be missing from `@shopify/polaris-types` — `@ts-expect-error` it.)

## 4. Polaris React → Web Components migration cheat-sheet

| Raw HTML | Polaris Web Component |
|---|---|
| hidden `<input type=checkbox>` + `.switch` label | `<s-switch label checked={v \|\| undefined} onChange>` |
| `<input type="color">` | `<s-color-field label name value onChange>` |
| +/- steppers + `<input type=number>` | `<s-number-field min max step suffix value={String(v)}>` |
| light/dark `<span>` toggles | `<s-choice-list><s-choice value selected={v \|\| undefined}>` |
| hardcoded color status pill | `<s-badge tone={ok?"success":"warning"}>` |
| `<a style={color}>` | `<s-link>` |
| inline flex `<div>` | `<s-stack direction="inline" justifyContent="space-between">` |

**Web-component rules learned**
- Boolean attrs: `attr || undefined` — passing `false` still renders the attribute.
- `value` is stringified (`value={String(borderRadius)}`).
- `change`/`input` bubble as DOM events, so `onChange` works on fields; but custom events like `dismiss` need `addEventListener` on a ref (see `design-banners-mobile-a11y.md`).
- **Shadow DOM blocks external CSS** — you can't restyle the internals of `<s-badge>`/`<s-heading>`. Where you need (e.g.) mobile font scaling, use native elements with classes.

## 5. Layout architecture rule (Polaris React)

Tailwind for **layout** (branches); Polaris only as **leaf nodes**. Avoid Polaris *layout* components for new code (`BlockStack`, `InlineStack`, `Layout`/`Layout.Section`, `Box`, `InlineGrid`) — use `flex`/`grid` divs. No anonymous inline event handlers (wrap in `useCallback`). No inline `style={{}}` except genuinely runtime-dynamic values.

## 6. Brand color (if any)

Only via Polaris CSS custom properties in an overrides stylesheet, on **badges/decoration — never buttons**:
```css
:root { --p-color-text-brand:#7c3aed; --p-color-icon-brand:#7c3aed; }
```
