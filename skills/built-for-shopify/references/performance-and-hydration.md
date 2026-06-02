# Performance & Hydration — CLS, LCP, INP (the fix catalogue)

Covers BFS 2.1.1 (LCP ≤2.5s), 2.1.2 (CLS ≤0.1), 2.1.3 (INP ≤200ms), all P75 / 28 days — a **mandatory gate**. This file is the *fixes*. To **measure** correctly (field/RUM, not Lighthouse), see `measuring-web-vitals.md`. To **locate offenders from code**, see `diagnosing-lcp-cls-from-code.md`.

**Know your stack — the dominant risk differs:**
- **Client-only SPA** (webpack/Vite, no SSR): risks are LCP from a big un-split bundle + a blank-until-data root gate, and CLS from variable-height/late content. The **hydration section below does NOT apply** (no server HTML to mismatch). The SPA levers are: webpack `optimization.splitChunks: { chunks: 'all' }`, route-level `React.lazy()`/`Suspense`, prod minification (`mode:'production'`), and **never gate first paint on an auth/data round-trip — paint a skeleton shell first**. (React 18 `createRoot` over React 17 `ReactDOM.render` helps INP via automatic batching.)
- **SSR** (Remix / React Router 7): all of the below applies, especially the hydration section (mismatches discard SSR HTML and tank LCP).

## CLS — Cumulative Layout Shift

### 1. Skeletons must match real content pixel-for-pixel
The #1 CLS killer. A `h-[300px]` skeleton under `h-[500px]` content = a guaranteed 200px shift.

```tsx
{/* Map placeholder — MUST match DashboardVisitorMap h-[500px] */}
<div className="h-[500px] animate-pulse rounded-lg bg-gray-100" />
<div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
  <div className="h-[256px] animate-pulse rounded bg-gray-100" />
  <div className="h-[256px] animate-pulse rounded bg-gray-100" />
</div>
```
- Add a placeholder for **every** sub-element (map controls, chart rows, toolbars).
- Use Tailwind/class heights, not inline `style={{height}}`.
- Mark skeletons `aria-hidden="true"`; the real container carries `role="table"` + `aria-label`.
- In a card-based UI, wrap the skeleton in the **same card** so there's no "cardless flash."
- **Variable-height content** (tables whose row count is unknown at skeleton time): you can't match it pixel-for-pixel. Instead **reserve space** — give the container a `min-height` sized to the typical result, or render a fixed number of placeholder rows (e.g. 10) at the real row height. Once data arrives, render into that reserved box so it grows/shrinks at most by the delta, not from zero. Empty states should occupy the same reserved height.

### 2. Reserve image dimensions
```tsx
<img src={...} width={1040} height={500} />   // reserves aspect ratio before load
```

### 3. Late-appearing banners stay `position: fixed`
A banner rendered inside `ClientOnly` (appears after an async query resolves) must use `fixed top-0 left-0 right-0`. Switching to flow layout pushes `<Outlet/>` down on resolution = visible CLS.

## LCP — Largest Contentful Paint

- **Parallelize independent loader calls:** `await Promise.all([...])`.
- **Query only the fields you need.** Replacing a 44-field `getFullShopJson` with a 4-field `getShopIdentity` on the plan path cut server time materially.
- **Defer heavy charts off the LCP path** with `lazy()` + `Suspense` inside a collapsible section:
  ```tsx
  const Chart = lazy(() => import("./Chart"));
  <Suspense fallback={<ChartSkeleton height={300} />}><Chart/></Suspense>
  ```
- **Streaming SSR** helps: React Router 7 / Remix `renderToPipeableStream` with bot-aware `onAllReady`/`onShellReady`.
- **Shed bundle weight:** dropping Polaris-React + Emotion + MUI (moving to Polaris Web Components) removed large client JS and directly improved CWV.

## Kill SSR hydration mismatches (they tank LCP)

A hydration mismatch makes React discard the SSR HTML and re-render client-side — hurting LCP. Causes seen and fixed:

1. **Locale/`Intl` formatting.** `toLocaleDateString()` differs server vs client. Use deterministic static-array helpers on UTC:
   ```ts
   const MONTHS_SHORT = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];
   export const formatShortDate = (d: Date) => `${MONTHS_SHORT[d.getUTCMonth()]} ${d.getUTCDate()}`;
   ```
   (Use `getUTCDay()`/`getUTCMonth()` to avoid timezone drift too.)
2. **`Date.now()` / `new Date()` / `Math.random()` at render time** (including inside `useRef`). Move out of render.
3. **Feature flags that differ SSR↔client.** Remove from the render path.
4. **Chrome auto-translate.** Add `lang="en"` to `<html>` — eliminated 100% of those errors in production (EXP-011).
5. **Volatile data tables:** drop SSR prefetch entirely; fetch client-side with React Query. One app removed `dehydrate`/`HydrationBoundary` from `/ips` and fell to client-only fetch, targeting an error-rate drop from 248% → <50%.
6. **`ClientOnly` wrapper** for any non-deterministic subtree (relative timestamps, etc.):
   ```tsx
   export function ClientOnly({ children, fallback = null }) {
     const [mounted, setMounted] = useState(false);
     useEffect(() => setMounted(true), []);
     return <>{mounted ? children : fallback}</>;
   }
   ```
7. **Log what slips through:** a `HydrationErrorBoundary` using `onRecoverableError` → analytics, capturing `componentStack`.

## INP — Interaction to Next Paint (≤200ms)

- Avoid synchronous heavy work in event handlers; defer with `lazy()`/code-splitting.
- Collapse expensive sections by default so their JS isn't parsed until opened.
- Keep handlers memoized (`useCallback`) and avoid re-rendering large trees on every keystroke (debounce dirty-tracking on a wrapper via `onChangeCapture`/`onInputCapture`).

## Verify

Use the `shopify-debug` web-vitals meta tag and/or a native `PerformanceObserver` piped to analytics (PostHog). BFS measures **P75 over a rolling 28 days** — a single fast load isn't enough; fix the systemic causes above.
