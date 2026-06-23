# Type Safety

TypeScript types must match API contracts. Runtime validation belongs at the API boundary.

## Type Organization

- Shared API DTOs live in `@unfreq/contracts`.
- Use subpath imports when they make ownership clearer: `@unfreq/contracts/api`, `@unfreq/contracts/auth`, `@unfreq/contracts/signals`, `@unfreq/contracts/admin`, and `@unfreq/contracts/theme-radio`.
- Frontend-only view models live near components or under `types/`.
- Do not import Drizzle table types into React components.
- `@/types/core` may contain frontend-only view types. It must not redefine API DTOs that exist in `@unfreq/contracts`.
- Theme radio DTOs come from `@unfreq/contracts/theme-radio` or the root `@unfreq/contracts` export because theme radio is a current galaxy hint.
- `@/types/legacy` and `@/types/product` are retired. New code must not recreate
  those barrels or import from them.

## Validation

- Backend validates request bodies, params, and query.
- Frontend validates forms for user feedback, but backend validation is authoritative.
- Selected Web API responses must pass a runtime validator at `apiClient` call sites before typed data enters UI state.
- AI responses must be parsed as structured JSON before use.
- Match, embedding, galaxy, recap, and share-card data must come from typed API DTOs.

## Common Patterns

```typescript
import type { ApiErrorResponse, ApiResponse } from "@unfreq/contracts/api";
import type { SignalDetail } from "@unfreq/contracts";
```

## Forbidden Patterns

- No `any` for API responses.
- No unchecked type assertions for API data.
- No direct `payload.data as T` trust path for core session, signal, galaxy, resonance, save, or echo response helpers.
- No raw database row types in UI props.
- No stringly-typed signal status, audio status, source, or frequency code formatting outside shared helpers.
- No UI-only types that invent fields absent from the API contract.
- No app-local copies of public API DTOs already exported by `@unfreq/contracts`.
- No import from retired legacy DTO groups or the old compatibility
  `@/types/product` barrel.

## Scenario: Web API Response Runtime Validation

### 1. Scope / Trigger

- Trigger: adding or changing Web helpers that call Fastify JSON APIs through `apiClient`.
- Applies to `apps/web/src/lib/api-client.ts`, response validators under
  `apps/web/src/lib/`, and typed API helper modules such as `signal-data.ts`.
- Does not change backend JSON response contracts.

### 2. Signatures

```typescript
type ApiResponseValidator<TData> = (value: unknown) => value is TData;

apiClient.get<TData>(path, { validate?: ApiResponseValidator<TData> })
apiClient.post<TData, TBody>(path, body, { validate?: ApiResponseValidator<TData> })
apiClient.patch<TData, TBody>(path, body, { validate?: ApiResponseValidator<TData> })
apiClient.delete<TData, TBody>(path, { body, validate?: ApiResponseValidator<TData> })
```

### 3. Contracts

- `apiClient` still validates the API envelope first: `{ data, meta.requestId }`.
- If `validate` is present, it checks `payload.data`, not the whole envelope.
- Validation failure throws `ApiClientError` with `code="INVALID_RESPONSE"`,
  the HTTP status, `requestId` when available, and `details.path`.
- Validators should import DTO types from `@unfreq/contracts` and reuse exported
  contract constants such as `signalPrimaryBands`.
- Keep validators at the Web API boundary. Components should receive typed data,
  not raw `unknown`.

### 4. Validation & Error Matrix

- Fetch rejects before response -> `NETWORK_ERROR`, unchanged.
- Non-2xx typed API error -> backend error code/message, unchanged.
- 2xx response missing API envelope -> `INVALID_RESPONSE`.
- 2xx response with envelope but invalid `data` -> `INVALID_RESPONSE`.
- 2xx response with no `validate` option -> legacy unchecked cast remains allowed only for un-migrated helpers.

### 5. Good/Base/Bad Cases

- Good: `getSignalGalaxy()` passes `validate: isGalaxyResponse`.
- Good: `/api/me` helpers pass `validate: isMeResponse` before returning a frequency.
- Base: a not-yet-migrated low-risk helper still relies on the envelope check until a scoped task migrates it.
- Bad: a page calls `apiClient.get<SignalResponse>()` directly and feeds the result into UI state without a validator.

### 6. Tests Required

- Unit test: valid data plus validator returns typed data.
- Unit test: invalid data plus validator throws `ApiClientError` with `INVALID_RESPONSE`.
- Unit test: backend HTTP/API errors bypass data validation and preserve the original error code.
- Run `npm run web:lint` and `npm run web:typecheck` after adding validators.

### 7. Wrong vs Correct

#### Wrong

```typescript
const result = await apiClient.get<SignalResponse>("/api/signals/123");
return result.signal;
```

#### Correct

```typescript
const result = await apiClient.get<SignalResponse>("/api/signals/123", {
  validate: isSignalResponse,
});
return result.signal;
```

## Required Enums

The UI should recognize these current product enums:

```text
SignalStatus: DRAFT, PUBLISHED, REVIEWING, HIDDEN, DELETED
SignalGenerationStatus: RECEIVED, EXPANDING, MATCHING, WRITING, COMPLETED, FAILED
SignalAudioStatus: NOT_REQUESTED, PENDING, READY, FAILED
SignalSource: human, seed, ai_generated
```

Legacy enums such as channel, voice, bottle, translation risk, or daily radio report may remain only in archived docs or backend historical schemas. Current UI must not depend on them.

## Frequency Identity

Frequency identity view models must match `/api/me`, `/api/frequencies`, and auth responses:

```text
FrequencyIdentity:
  id: string
  code: string              // display format: FM 78.23
  rawCode?: string          // optional sortable/storage code if API exposes it
  frequencyName?: string
  isGuest?: boolean
  role: ADMIN | USER
  status: ACTIVE | MUTED | BANNED
  lastSeenAt?: ISO datetime
  createdAt: ISO datetime
```

Guest-first UI must treat `code` as the public display identity and `frequencyName` as an optional saved-login handle. Do not require email in new first-use flows.

## Signal DTOs

Signal view models must match `/api/signals/*` responses:

```text
SignalSummary:
  id: string
  frequencyCode: string
  seedText: string
  expandedText: string
  resonanceCount: number
  hasResonated?: boolean
  source: human | seed | ai_generated
  status: SignalStatus
  createdAt: ISO datetime

SameFrequencyMatch:
  id: string
  frequencyCode: string
  seedText?: string
  expandedText: string
  resonanceCount: number
  distance?: number
  source: PUBLISHED | SEEDED | FALLBACK

SignalDetail:
  id: string
  frequencyCode: string
  seedText: string
  expandedText: string
  followUpQuestion?: string
  parentSignalId?: string
  matches: SameFrequencyMatch[]
  audio: SignalAudioDetail
  resonanceCount: number
  hasResonated: boolean
  hasSaved?: boolean
  source: human | seed | ai_generated
  status: SignalStatus
  createdAt: ISO datetime
  updatedAt: ISO datetime

SignalAudioDetail:
  signalId: string
  status: SignalAudioStatus
  audioUrl?: string
  error?: string

ReviewingSignalRestrictedDetail:
  id: string
  status: REVIEWING
  visibility: REVIEW_RESTRICTED
  createdAt: ISO datetime
  updatedAt: ISO datetime

SignalPreviewDetail:
  SignalDetail fields
  status: DRAFT
  visibility: PREVIEW_OWNER

SignalGenerationDetail:
  id: string
  status: SignalStatus
  generationStatus: SignalGenerationStatus
  seedText: string
  parentSignalId?: string
  generationError?: string
  visibility: GENERATION_OWNER
  createdAt: ISO datetime
  updatedAt: ISO datetime

SignalResultDetail:
  SignalDetail | SignalPreviewDetail | ReviewingSignalRestrictedDetail | SignalGenerationDetail

PublishSignalResponse:
  signal: SignalDetail
```

`expandedText` is the primary reading and playback text. `seedText` is context only and must not be used for embedding or TTS unless a fallback explicitly says so.
Display labels should use `fingerprint.tags` when present. `frequencyProfile.resonanceKeywords` are matching hints and should be used as a fallback or metadata context, not as the primary human label source.
`GET /api/signals/:id` can return `ReviewingSignalRestrictedDetail` for
non-owner or anonymous access to a reviewing signal. Result pages must branch on
`visibility === "REVIEW_RESTRICTED"` before reading `seedText`, `expandedText`,
`fingerprint`, matches, audio, resonance, save, or share fields. Server-rendered
result pages must forward the request cookie to the API so the owner receives
the owner reviewing state instead of anonymous restricted state.
`POST /api/signals` returns `SignalGenerationDetail`, not full `SignalDetail`.
Web must route that id to `/signal/:id/transmitting` and poll
`GET /api/signals/:id` until `generationStatus === "COMPLETED"`. Generating or
failed rows are owner-only. Result pages that receive
`visibility === "GENERATION_OWNER"` must redirect back to the transmitting page
instead of reading full-detail fields.
Completed owner previews use `visibility === "PREVIEW_OWNER"`. Result pages may
read seed and expanded text for the owner preview, but must not render public
matches, resonance, save, playback, share card, or galaxy actions until
`publishSignal()` returns a `PUBLISHED` `SignalDetail`.

## Scenario: Web Pending Signal Generation Contract

### 1. Scope / Trigger

- Trigger: changing `createSignal()`, `getSignal()`, signal response validators,
  `/signal/:id/transmitting`, or `/result/:id` generation handling.
- Applies to Web because the same signal detail endpoint returns completed,
  reviewing-restricted, and owner-only generation variants.

### 2. Signatures

```typescript
createSignal(input: CreateSignalInput): Promise<SignalGenerationDetail>;
publishSignal(signalId: string): Promise<SignalDetail>;
getSignal(id: string, cookieHeader?: string): Promise<SignalResultDetail | undefined>;
isSignalGenerationDetail(signal: SignalResultDetail): signal is SignalGenerationDetail;
isSignalPreviewDetail(signal: SignalResultDetail): signal is SignalPreviewDetail;
isSignalDetail(signal: SignalResultDetail): signal is SignalDetail;
```

```text
CreateSignalInput:
  seedText: string
  parentSignalId?: string

CreateSignalResponse:
  signal: SignalGenerationDetail

SignalResponse:
  signal: SignalDetail | SignalPreviewDetail | ReviewingSignalRestrictedDetail | SignalGenerationDetail

PublishSignalResponse:
  signal: SignalDetail
```

### 3. Contracts

- `createSignal()` must validate `CreateSignalResponse` and return the
  `SignalGenerationDetail`.
- Homepage submit must navigate to `/signal/:id/transmitting`, not `/result/:id`.
- The transmitting page must poll `getSignal()` with cookies so owner-only
  generation rows remain readable after refresh.
- `GENERATION_OWNER` must be branched before reading `expandedText`,
  `fingerprint`, matches, audio, resonance, save, or share fields.
- `COMPLETED` generation redirects to `/result/:id`; low/medium-risk completed
  rows render the private owner preview there until publish.
- `PREVIEW_OWNER` must be branched before public actions. It may show seed and
  expanded text plus a publish action, but must hide matches, playback,
  resonance, save, and share-card UI.
- `publishSignal()` must validate `PublishSignalResponse` and return only a
  `SignalDetail` whose status is `PUBLISHED`.
- `FAILED` generation stays on transmitting and may offer retry by creating a
  new signal with the same seed.
- Public result, random, galaxy, match, save, resonance, and share-card
  consumers must ignore generation and preview variants.

### 4. Validation & Error Matrix

- Valid create response with `GENERATION_OWNER` -> return id for transmitting.
- Valid detail response with `SignalDetail` -> render result.
- Valid detail response with `PREVIEW_OWNER` -> render private preview and
  publish action only.
- Valid detail response with `REVIEW_RESTRICTED` -> render restricted result.
- Valid detail response with `GENERATION_OWNER` on transmitting -> render or
  redirect by `generationStatus`.
- `GENERATION_OWNER` on result page -> redirect to transmitting.
- Publish preview response with `SignalDetail.status="PUBLISHED"` -> refresh
  result page.
- Publish invalid, non-owned, or incomplete signal -> backend error preserved.
- Backend `400` or `404` in `getSignal()` -> return `undefined`.
- Invalid generation status or missing `visibility` -> `INVALID_RESPONSE`.

### 5. Good/Base/Bad Cases

- Good: submit seed, receive `RECEIVED`, navigate to transmitting, poll until
  `COMPLETED`, then redirect to result.
- Good: page refresh during `EXPANDING` still shows transmitting because cookies
  are forwarded to the API.
- Good: owner sees `PRIVATE PREVIEW` on result, confirms publish, then sees the
  normal public result page.
- Base: backend returns `FAILED`; UI keeps the seed and lets the user retry.
- Bad: cast `SignalResponse.signal as SignalDetail` and read `expandedText`.
- Bad: treat `PREVIEW_OWNER` as `SignalDetail` and render resonance, save,
  playback, or share-card controls before publish.
- Bad: send new signals straight to `/result/:id` and rely on the result page to
  display pending state.

### 6. Tests Required

- Web typecheck: `SignalResultDetail` consumers branch before full-detail field
  access.
- Validator test or focused check: `isCreateSignalResponse` accepts
  `SignalGenerationDetail` and rejects missing `generationStatus`.
- Validator test or focused check: `isSignalResponse` accepts
  `SignalPreviewDetail`, and `isPublishSignalResponse` rejects non-published
  details.
- Route/page test: result page redirects generation rows to transmitting.
- E2E test: seed emission reaches private preview, confirm publish, then public
  same-frequency and galaxy navigation appear.
- UI check: transmitting page handles `RECEIVED`, `EXPANDING`, `MATCHING`,
  `WRITING`, `COMPLETED`, and `FAILED`.

### 7. Wrong vs Correct

#### Wrong

```typescript
const signal = await createSignal(input);
router.push(`/result/${signal.id}`);
```

#### Correct

```typescript
const signal = await createSignal(input);
router.push(`/signal/${signal.id}/transmitting`);
```

#### Wrong

```typescript
if (isSignalDetail(signal)) {
  return <PublicActions signal={signal} />;
}
```

#### Correct

```typescript
if (isSignalPreviewDetail(signal)) {
  return <PreviewPublishOnly signal={signal} />;
}

if (isSignalDetail(signal)) {
  return <PublicActions signal={signal} />;
}
```

## Galaxy DTOs

Galaxy view models must match `GET /api/signals/galaxy`:

```text
GalaxySignalPoint:
  id: string
  frequencyCode: string
  expandedText: string
  resonanceCount: number
  source: human | seed | ai_generated
  theme: string
  isMine: boolean
  position:
    x: number
    y: number
    z: number
  matchIds: string[]
  createdAt: ISO datetime

GalaxyResponse:
  points: GalaxySignalPoint[]
  generatedAt: ISO datetime
  themeOptions?: string[]
```

Coordinates must be finite numbers. The frontend may normalize them for camera framing, but must not compute embedding or semantic layout.
`matchIds` are ids of other points already present in the same response. The frontend must ignore missing ids rather than fetching or computing matches locally.
Galaxy pages should call the typed `getSignalGalaxy()` API helper and pass through cookies when server-rendered so `isMine` can be computed by the API.
Theme filters should pass only a trimmed 1-48 char `theme` query. Invalid or overlong theme query values should be ignored before calling the API.
When the API returns `themeOptions`, Web should use it for the filter row instead
of issuing a second unfiltered galaxy request. A fallback may derive options from
the returned points only for compatibility with older API responses.
When `points` changes after a theme switch, reset the selected point from the new point set; do not keep a detail sheet for a point outside the filtered response.

## Saves, Recaps, and Theme Radio DTOs

Signal saves must match the signal save and my-save APIs:

```text
SignalSaveResponse:
  save:
    hasSaved: boolean

SavedSignalSummary:
  id: string
  seedText: string
  expandedText: string
  parentSignalId?: string
  resonanceCount: number
  status: SignalStatus
  createdAt: ISO datetime
  savedAt: ISO datetime
  theme: string
  authorFreqCode?: string

SavedSignalsResponse:
  signals: SavedSignalSummary[]
```

Save buttons may optimistically reflect only the returned `hasSaved`. They must show a failure state for unauthorized, missing, or failed mutations and must not invent author notifications.

Monthly recap view models must match `/api/me/recaps/*`:

```text
MonthlyRecap:
  id: string
  frequencyId: string
  yearMonth: YYYY-MM
  signalCount: number
  totalResonanceCount: number
  summary: string
  themeWords: string[]
  topSignal?: MySignalSummary
  generatedAt: ISO datetime
  createdAt: ISO datetime
  updatedAt: ISO datetime

MonthlyRecapResponse:
  recap: MonthlyRecap

LatestMonthlyRecapResponse:
  recap?: MonthlyRecap
```

`/me` must treat missing latest recap as a real empty state. It must not fabricate a report from local data.

Theme radio view models must match `/api/theme-radio/current` and admin theme endpoints:

```text
ThemeRadioDay:
  id: string
  theme: string
  title: string
  description?: string
  startsAt: ISO datetime
  endsAt?: ISO datetime
  isActive: boolean
  createdAt: ISO datetime
  updatedAt: ISO datetime

ThemeRadioDayResponse:
  themeRadioDay?: ThemeRadioDay
```

Theme radio UI should be a light hint that links to `/galaxy?theme=<theme>`. It must not render a hot-topic feed, ranking, or recommendation list.

## Echo Summary DTOs

Return-visit resonance prompts must match `GET /api/me/echo-summary`:

```text
MySignalSummary:
  id: string
  seedText: string
  expandedText: string
  parentSignalId?: string
  resonanceCount: number
  status: SignalStatus
  createdAt: ISO datetime

EchoSummary:
  frequencyId: string
  newResonanceCount: number
  totalResonanceCount: number
  since?: ISO datetime
  topSignal?: MySignalSummary
```

`POST /api/me/echo-summary/seen` returns the same `EchoSummary` shape after advancing the backend watermark.

`parentSignalId` marks continued signals on `/me`. My-frequency UI may build a local thought tree from the returned list, but it must tolerate a missing parent item and still render the child.

Do not invent resonance totals in the frontend. If `since` is absent and the API returns `newResonanceCount=0`, Web may compare `totalResonanceCount` against a browser-local previously seen total for the same `frequencyId`; that derived prompt is UI-local only and must not be sent back as an API truth. Empty summaries render no prompt or a quiet empty state. When a visible prompt is rendered or dismissed, call the seen endpoint and update browser-local seen totals so the same batch is not shown repeatedly.

## Share Card DTOs

Share-card preview data must be derived from public signal detail:

```text
ShareSignalCard:
  signalId: string
  imageUrl: string
  frequencyCode: string
  expandedText: string
```

Generated card images must not expose raw database ids other than public signal ids already present in routes.
The image endpoint is Web-owned at `/api/og/:signalId`; it must reuse the public
signal detail contract, return PNG for `PUBLISHED` signals, and return 404 when
the public detail API returns 404 or a `PREVIEW_OWNER`/generation variant.

## Admin DTOs

Admin pages must use typed DTOs from Fastify admin endpoints. They must not invent
local fields from old voices/bottles data as the primary dashboard model.

```text
AdminOverview:
  signals:
    total: number
    published: number
    reviewing: number
    hidden: number
    deleted: number
    draft: number
  reports:
    pending: number
    resolved: number
    rejected: number
  frequencies:
    active: number
    muted: number
    banned: number
  aiOperations:
    chatFallbacks: number
    embeddingFailures: number
    rerankerFailures: number
    ttsFailures: number
  capabilities:
    chat: configured | degraded | disabled
    embedding: configured | degraded | disabled
    reranker: configured | degraded | disabled
    tts: configured | degraded | disabled
  galaxy:
    missingCoordinates: number
```

Provider config admin DTOs must match `GET /api/admin/provider-configs`,
`POST /api/admin/provider-configs/:kind`,
`PUT /api/admin/provider-configs/:kind/:id`,
`DELETE /api/admin/provider-configs/:kind/:id`, and legacy
`PUT /api/admin/provider-configs/:kind`:

```text
AdminProviderConfigKind:
  chat | embedding | tts | reranker

AdminProviderConfigRecord:
  id?: string
  kind: AdminProviderConfigKind
  provider: string
  model: string
  baseUrl: string
  isEnabled: boolean
  priority: number
  source: database | environment
  status: configured | degraded | disabled
  keyState: configured | missing
  apiKeyLast4?: string
  extraJson: Record<string, unknown>
  updatedAt?: ISO datetime
  updatedBy?: string

AdminProviderConfigUpdate:
  provider: string
  model: string
  baseUrl: string
  kind?: AdminProviderConfigKind
  isEnabled?: boolean
  priority?: number
  apiKey?: string
  clearApiKey?: boolean
  extraJson?: Record<string, unknown>
```

Provider config forms must treat `apiKey` as write-only. They may display
`apiKeyLast4` and `keyState`, but must never expect a full API key from the API.
Rows with `source=environment` have no `id`; saving them must create a database
backup row through `POST`, not overwrite another database row for the same kind.
Rows with `source=database` must update/delete by `kind + id`.
Environment rows may display `priority=1000`, but Web must never send that value
back in provider config write payloads. When a user saves an environment row,
the form should submit a writable database priority inside `1..999` instead.
Embedding-specific controls map to `extraJson.dimensions` and
`extraJson.version`. TTS voice controls map to `extraJson.voiceLeft` and
`extraJson.voiceRight`.

```text
AdminSignalRecord:
  id: string
  frequencyCode?: string
  seedText: string
  expandedText: string
  source: human | seed | ai_generated
  status: SignalStatus
  resonanceCount: number
  audioStatus: SignalAudioStatus
  embedding:
    status: ready | missing
    provider?: string
    model?: string
    dimensions?: number
    version?: string
  galaxyPositionStatus: ready | missing
  qualityScore?: number
  qualityReason?: string
  createdAt: ISO datetime
  updatedAt: ISO datetime
```

Admin signal status mutations call `PATCH /api/admin/signals/:id/status`.
Seed-library keep/delete actions may still call `/api/admin/ai-signals/*`.
Frequency status mutations call `PATCH /api/admin/frequencies/:id/status`, not
the legacy `/api/admin/users/:id/status`, when the UI label is FM frequency.

## Legacy Types

Expression translation, risk radar, voices, bottles, echoes, channel DTOs, `@/types/legacy`, and `@/types/product` are retired for the current Unfreq direction.

Do not restore them as compatibility barrels. If a future task revives one retired product surface, create a fresh explicit DTO file for that task and add a matching API contract first.
