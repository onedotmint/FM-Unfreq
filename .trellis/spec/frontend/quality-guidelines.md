# Quality Guidelines

Frontend quality means product consistency, accessibility, responsive layout, stable API use, and a nonblank galaxy experience.

## Required Patterns

- Use current Unfreq terms consistently.
- Use `apiClient` for all backend calls.
- Keep core navigation focused on: entry, galaxy, my frequency. Result pages are reached from signal actions.
- Protect admin pages in both UI and API. UI hiding alone is not security.
- Show pending and failure states for mutations.
- Preserve seed input on submit failure.
- Empty or low-content states must explain what is actually empty and provide a real next action.
- `/galaxy` changes require desktop and mobile visual checks.
- Playback must be click/tap initiated.

## Tooling Conventions

### Convention: Next 16 ESLint Flat Config

**What**: Import Next flat configs directly in `eslint.config.mjs`.

**Why**: Wrapping `next/core-web-vitals` with `FlatCompat` can produce circular config validation errors with ESLint 9.

**Example**:

```typescript
import nextCoreWebVitals from "eslint-config-next/core-web-vitals";
import nextTypescript from "eslint-config-next/typescript";

export default [
  ...nextCoreWebVitals,
  ...nextTypescript,
];
```

### Convention: Next Env Type Drift

**What**: Treat `apps/web/next-env.d.ts` as a generated file. Do not keep
incidental drift from local dev runs, especially `./.next/dev/types/routes.d.ts`
in unrelated work.

**Why**: Next can rewrite this file during local commands. Committing that drift
mixes generated environment state into feature, fix, or docs commits.

**Correct Baseline**:

```typescript
import "./.next/types/routes.d.ts";
```

**Tests Required**:
- Run `npm run web:lint` and `npm run web:typecheck` after restoring this file.

### Convention: Service Worker Network-First App Assets

**What**: The service worker may precache `/offline`, but normal GET assets and
RSC/script chunks must try network first and fall back to cache only when the
network fails.

**Why**: Stale cached application files can keep old auth UI alive after a
release, including legacy auto-guest flows.

**Example**:

```javascript
event.respondWith(fetch(request).catch(() => caches.match(request)));
```

**Tests Required**:
- `web:build`, `web:typecheck`, and `web:lint` must pass after service worker changes.
- Manual browser verification should unregister or update the old worker when debugging stale UI.

### Convention: Admin Server-Paginated Lists

**What**: Admin tables that can grow with product data must keep pagination,
search, and filters at the API boundary.

**Why**: Fetching only the newest page and then filtering in React hides older
matching records. Fetching every row makes the admin dashboard degrade as signal
volume grows.

**Contracts**:
- Signal governance and seed-library tables call the admin API whenever
  `limit`, `cursor`, `q`, `source`, or `status` changes.
- Search input becomes a query parameter, not an in-memory filter over the
  current page.
- Components keep only the current page, loading/error state, selected cursor,
  and previous cursors needed for navigation.
- Page size respects the backend contract: default `50`, maximum `100`.
- Web code treats `nextCursor` as opaque and sends it back unchanged.

**Tests Required**:
- Typecheck after changing admin API helpers or shared admin contracts.
- Component/manual check that search and page changes trigger API fetches.

**Wrong**:

```typescript
const visibleSignals = signals.filter((signal) => signal.seedText.includes(q));
```

**Correct**:

```typescript
const page = await fetchAdminSignals({ limit, cursor, q, source, status });
```

## Forbidden Patterns

- No "like/comment/post/user/profile" copy in visible UI.
- No "voice/bottle/channel hall/risk radar/translation" copy in new core UI.
- No direct AI provider calls.
- No database imports in frontend.
- No frontend-only moderation decisions.
- No hot list, ranking, follower, or recommendation UI.
- No fake activity counts or primary actions that are known to fail only to make an empty state feel alive.
- No autoplay audio.
- No card-heavy feed as a replacement for `/galaxy`.
- No "all admin signals" table derived from a fixed first page.

## Empty State Contracts

### Convention: Actionable Low-Content States

**What**: When a server-rendered product surface has no items, render a stable empty state with:

1. The product object that is missing, such as signals or same-frequency matches.
2. One truthful reason or boundary, such as no published signals or no new resonance since last visit.
3. A next action that can succeed now, such as write a seed, tune randomly, return home, or open my frequency.

**Why**: Low-content periods should feel quiet, not broken. The UI must not imply live activity that the API did not return.

**Example**:

```typescript
if (!galaxy.points.length) {
  return "星图还没有足够的信号。先写下一句种子。";
}
```

**Wrong**: Showing fake resonance counts or fake live users to fill an empty galaxy.

**Correct**: Use seed content marked as `source='seed'`, or show a truthful low-content state with a valid action.

## Scenario: SSR API Unavailable Fallbacks

### 1. Scope / Trigger

- Trigger: server-rendered product pages need to stay reviewable when the local API is not running.
- Applies to core SSR routes such as `/`, `/galaxy`, and `/me`.
- Does not apply to mutations that create, update, resonate, save, or publish signals.

### 2. Signatures

```typescript
apiClient.get<TData>(path, { timeoutMs?: number })
apiClient.post<TData, TBody>(path, body, { timeoutMs?: number })
isApiUnavailableError(error: unknown): boolean
```

### 3. Contracts

- `timeoutMs` is optional and should be used only where a short read timeout improves reviewability or page resilience.
- Network failures before an HTTP response must become `ApiClientError` with `code: "NETWORK_ERROR"` and `status: 0`.
- SSR pages may render a controlled unavailable state only when `isApiUnavailableError(error)` is true.
- Optional SSR data loaded through `Promise.allSettled` must keep variables typed as `T | undefined`.
  Do not pass possibly unassigned `let` variables into components after an unavailable branch.

### 4. Validation & Error Matrix

- API returns `401` -> keep existing unauthenticated or guest state.
- API returns `400`, `403`, `404`, `500` -> preserve existing route-specific behavior; do not treat it as local downtime.
- Fetch rejects or times out before response -> render a truthful unavailable state for core SSR pages.
- Unknown programming error -> rethrow to route error boundary.

### 5. Good/Base/Bad Cases

- Good: `/galaxy` shows "星图暂时没有接入" when the local API is down.
- Base: `/galaxy` shows "星图还没有信号" when the API returns an empty points array.
- Bad: swallowing all errors and showing an empty galaxy for a backend 500 or invalid DTO.

### 6. Tests Required

- Typecheck must verify the `timeoutMs` option and `isApiUnavailableError` call sites.
- Lint must pass without suppressed warnings.
- Visual checks should capture `/`, `/galaxy`, `/me`, and `/offline` when this behavior changes.

### 7. Wrong vs Correct

#### Wrong

```typescript
try {
  return await getSignalGalaxy();
} catch {
  return { points: [] };
}
```

#### Correct

```typescript
try {
  return await getSignalGalaxy();
} catch (error) {
  if (!isApiUnavailableError(error)) {
    throw error;
  }

  return { isUnavailable: true, points: [] };
}
```

## Scenario: Optional SSR Section Degradation

### 1. Scope / Trigger

- Trigger: server-rendered product pages combine critical identity/list data with
  optional sections such as recap, saved signals, notices, or secondary summaries.
- Applies to core SSR routes such as `/me`.
- Does not apply to mutations or primary page data required to understand the
  route.

### 2. Signatures

```typescript
Promise.allSettled([loadOptionalA(), loadOptionalB()])
isApiUnavailableError(error: unknown): boolean
error instanceof ApiClientError
```

### 3. Contracts

- Load critical route data first. If it fails, render the route-level controlled
  state or rethrow according to the route contract.
- Load optional sections through `Promise.allSettled`.
- Optional sections may degrade only to truthful empty or unavailable content.
- Do not make optional fallback data look like real server data.
- On `/me`, current FM identity is the route-critical data. Signal history,
  echo summary, recap, and saved signals may degrade independently once identity
  is available.

### 4. Validation & Error Matrix

- Critical `401` -> existing unauthenticated state.
- Critical network failure -> controlled unavailable state.
- Optional network failure -> omit that section or render its empty contract.
- Optional `404` for an absent secondary resource -> omit that section or render
  its empty contract.
- `/me` identity loaded + signal history fails -> render the FM shell and a
  truthful sync-pending notice.
- Optional `500`, `403`, validation error, or unknown error -> rethrow.

### 5. Good/Base/Bad Cases

- Good: `/me` still renders frequency and signal history when latest recap is
  temporarily unavailable.
- Good: `/me` still renders the FM identity shell when `/api/me/signals` fails,
  with no fake signal rows.
- Base: `/me` renders "还没有月度回顾" when the recap response has no recap.
- Bad: `/me` throws to the global error page because saved signals failed while
  identity and signal history loaded.

### 6. Tests Required

- Typecheck verifies settled result narrowing.
- Lint passes without ignored promises or suppressed errors.
- Browser check verifies the page does not show the global error copy when an
  optional section returns a degradable failure.

### 7. Wrong vs Correct

#### Wrong

```typescript
const [profile, recap, saved] = await Promise.all([
  getProfile(),
  getLatestRecap(),
  getSavedSignals(),
]);
```

#### Correct

```typescript
const profile = await getProfile();
const [recapResult, savedResult] = await Promise.allSettled([
  getLatestRecap(),
  getSavedSignals(),
]);
```

## Testing Requirements

- Component tests for core UI states.
- Form tests for validation and submit failure preservation.
- With the current Vitest-only Web test stack, prefer extracting deterministic
  component state helpers for labels, disabled states, and error copy before
  adding a DOM test dependency. Keep the helper in the component directory and
  cover it with `*.test.ts`.
- Reusable inline action feedback should keep pending/success/failure state near
  the triggering action. Error feedback uses `role="alert"`; pending and success
  feedback use `role="status"` with polite live announcements. Prefer a small
  local helper/component over global toast infrastructure for copy, save,
  resonance, playback, and similar inline actions.
- Default local verification should use `npm run web:lint`,
  `npm run web:typecheck`, and `npm --workspace @unfreq/web run test`.
- Do not run `npm run e2e` by default. It is intentionally opt-in because
  Next dev cold start makes the command slow in local agent sessions.
- E2E smoke tests remain available for create signal, result view, resonance,
  galaxy render, and my frequency, but run them only when the user explicitly
  asks for E2E or when a change cannot be checked below the browser layer.
- Responsive checks for text overflow, input layout, bottom sheets, and playback buttons.
- Playwright screenshot or canvas-pixel check for `/galaxy` after Three.js changes.

### Convention: Local Mock API E2E Harness

**What**: `npm run e2e` runs Playwright against the Next.js web app and a local
mock API server. The web server must point at the mock API with both
`NEXT_PUBLIC_API_BASE_URL` and `API_INTERNAL_BASE_URL`, so SSR and browser
requests see the same fixture data.

**Why**: Core pages load data on the server before hydration. Browser-only
request interception is not enough to cover that boundary.

**Gotcha**: Next dev may rewrite generated TypeScript route references while
the Playwright server is running. After local e2e, check `apps/web/next-env.d.ts`
and `apps/web/tsconfig.json`; do not commit `.next/dev/...` churn unless a
tooling task intentionally changes the Next type-generation contract.

**Example**:

```typescript
webServer: [
  {
    command: "node tests/e2e/mock-api.mjs",
    env: {
      E2E_API_PORT: "4100",
      E2E_WEB_ORIGIN: "http://127.0.0.1:3100",
    },
  },
  {
    command: "node_modules/.bin/next dev apps/web --hostname 127.0.0.1 --port 3100",
    env: {
      API_INTERNAL_BASE_URL: "http://127.0.0.1:4100",
      NEXT_PUBLIC_API_BASE_URL: "http://127.0.0.1:4100",
    },
  },
]
```

**Tests Required**:
- Keep the E2E script separate from `npm run check`.
- Verify at least one seed-to-result-to-galaxy browser flow locally.
- Keep mock API fixture DTOs aligned with Web runtime validators. If a fixture
  response fails `api-response-validators.ts`, the browser flow should fail
  visibly instead of bypassing validation in tests.

### Convention: Galaxy Mobile Full-Bleed Check

**What**: After changing `/galaxy`, `RadioShell`, or mobile `.radio-main`
styles, verify that `.radio-main-galaxy`, `.galaxy-stage`, and the Three.js
canvas all measure the full viewport width on mobile.

**Why**: The generic mobile `.radio-main` rule narrows normal pages with
`calc(100vw - 24px)`. If that rule wins on galaxy pages, the full-bleed canvas
shrinks, the star map looks smaller, and click coordinates no longer match the
intended immersive layout.

**Check**:

```typescript
expect(canvas.getBoundingClientRect().left).toBe(0);
expect(canvas.getBoundingClientRect().width).toBe(window.innerWidth);
expect(document.documentElement.scrollWidth).toBe(document.documentElement.clientWidth);
```

## Code Review Checklist

- Does the page use the current product language?
- Does mobile layout fit without overlap?
- Are API calls using shared contracts?
- Are loading, empty, and error states present?
- Do empty states avoid fake activity and point to an action that can succeed?
- Is admin-only UI backed by API permission checks?
- Is audio user-initiated?
- Is `/galaxy` nonblank and full-bleed?
- Does `/galaxy` use the fixed seven-band filter, not old theme/tag filtering?
- Does manual rotate/zoom pause auto-rotation and then resume after idle?
