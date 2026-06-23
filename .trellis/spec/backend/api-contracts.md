# API Contracts

This project exposes JSON APIs from Fastify. Next.js Web and the Android Kotlin client must call the same contracts.

## Current Product Contract

The current Unfreq direction is anonymous thought resonance:

```text
seed text -> pending signal -> AI micro-monologue -> private preview -> owner publish -> same-frequency matches -> resonance -> galaxy
```

New core work must target the signal APIs in `Scenario: Pending Signal Emission API`. Older scenarios for expression translation, voice/channel/bottle surfaces, risk radar, and double-channel radio programs are retired product surfaces unless a task explicitly says to restore them.

## Scenario: Pending Signal Emission API

### 1. Scope / Trigger

- Trigger: changing `POST /api/signals`, signal creation service code, homepage
  emission, generation status polling, moderation status decisions,
  same-frequency matching during create, or embedding queueing.
- Applies to backend API and Web because the same Fastify JSON API must also
  serve future Android clients.

### 2. Signatures

```text
POST /api/signals
Body: { seedText: string, parentSignalId?: uuid }
Success: 201 { signal: SignalGenerationDetail }

GET /api/signals/:id
Generating owner -> 200 { signal: SignalGenerationDetail }
Generating non-owner/anonymous -> 404 NOT_FOUND
Completed DRAFT owner -> 200 { signal: SignalPreviewDetail }
Completed DRAFT non-owner/anonymous -> 404 NOT_FOUND
Completed PUBLISHED -> 200 { signal: SignalDetail }
Completed REVIEWING owner -> 200 { signal: SignalDetail }
Completed REVIEWING non-owner/anonymous -> 200 { signal: ReviewingSignalRestrictedDetail }

POST /api/signals/:id/publish
Success: 200 { signal: SignalDetail }

SignalGenerationStatus:
RECEIVED -> EXPANDING -> MATCHING -> WRITING -> COMPLETED
FAILED terminal on generation failure

SignalStatus after completion:
LOW/MEDIUM risk -> DRAFT private preview
HIGH risk       -> REVIEWING
Owner publish   -> PUBLISHED

SignalGenerationDetail:
  id: uuid
  status: SignalStatus
  generationStatus: SignalGenerationStatus
  seedText: string
  parentSignalId?: uuid
  generationError?: string
  visibility: GENERATION_OWNER
  createdAt: ISO datetime
  updatedAt: ISO datetime

SignalPreviewDetail:
  SignalDetail fields
  status: DRAFT
  visibility: PREVIEW_OWNER
```

Removed legacy generation path:

```text
POST /api/signals/generate
```

### 3. Contracts

- `POST /api/signals` is the normal product seed-emission API. It returns a
  stable signal id before AI generation finishes.
- Normal homepage emission creates a `signals` row with `status="DRAFT"` and
  `generationStatus="RECEIVED"`, then schedules generation for the same row.
- The background generation path updates the same row through
  `EXPANDING`, `MATCHING`, `WRITING`, then `COMPLETED`.
- Low- and medium-risk completed content stays `DRAFT` as a private owner
  preview until the owner confirms publish.
- `POST /api/signals/:id/publish` publishes only a completed `DRAFT` signal
  owned by the current active frequency. Calling it for an already `PUBLISHED`
  owned signal is idempotent and returns the public `SignalDetail`.
- High-risk completed content becomes `REVIEWING`. Clients should navigate from
  `/signal/:id/transmitting` to `/result/:id` and render the reviewing state
  through the shared detail API.
- Generation failures keep `status="DRAFT"` and set
  `generationStatus="FAILED"`. The owner sees a generic failure message and can
  create a new signal with the same seed; the failed row is not public.
- `GET /api/signals/:id` exposes `SignalGenerationDetail` only when the
  authenticated frequency owns the generating row.
- `GET /api/signals/:id` exposes `SignalPreviewDetail` only when the
  authenticated frequency owns the completed private preview.
- Non-owner and anonymous access to generating or failed rows returns
  `404 NOT_FOUND`; do not return restricted generation details to strangers.
- Non-owner and anonymous access to private previews returns `404 NOT_FOUND`.
- Public result, random, match, galaxy, save, resonance, and share-card surfaces
  must treat generating rows and private previews as invisible.
- Same-frequency matching is optional during creation. Matching failures degrade
  to an empty `matches` array and no `matchSignalId` after completion.
- Embedding write is queued once after generation completes. Pending placeholder
  text must not be embedded. Completed private previews may have embeddings, but
  public match reads must still filter to `PUBLISHED`.
- The current background runner is process-local. Persisted generation status
  survives page refresh and link reopening; cross-process restart recovery needs
  an explicit worker/queue task.
- `POST /api/signals/generate` is not registered in the current product API.
  Requests to that path return `404 NOT_FOUND` through route absence.

### 4. Validation & Error Matrix

- No session -> `401 UNAUTHORIZED`.
- Non-active frequency -> `403 FORBIDDEN`.
- Invalid `seedText` length or invalid `parentSignalId` -> `400 VALIDATION_ERROR`.
- Parent signal not visible to the same frequency -> `404 NOT_FOUND`.
- Accepted seed -> `201` with `SignalGenerationDetail`.
- Generating owner detail -> `200` with `SignalGenerationDetail`.
- Generating non-owner/anonymous detail -> `404 NOT_FOUND`.
- Completed low/medium-risk owner detail before publish -> `200` with
  `SignalPreviewDetail`.
- Completed low/medium-risk non-owner/anonymous detail before publish ->
  `404 NOT_FOUND`.
- Publish without session -> `401 UNAUTHORIZED`.
- Publish from non-active frequency -> `403 FORBIDDEN`.
- Publish missing or non-owned signal -> `404 NOT_FOUND`.
- Publish before generation completes -> `409 CONFLICT`.
- Publish `REVIEWING` signal -> `409 CONFLICT`.
- Publish completed owner preview -> `200` with `SignalDetail` and
  `status="PUBLISHED"`.
- AI material generation unavailable after fallback -> generating row becomes
  `FAILED`; later owner `GET` returns `SignalGenerationDetail`.
- Same-frequency matching failure -> completed signal has `matches: []`.
- Embedding queue failure after completion -> logged warning, completed signal
  response remains successful.
- Legacy draft generation path -> `404 NOT_FOUND`.

### 5. Good/Base/Bad Cases

- Good: homepage submit calls `POST /api/signals`, receives
  `SignalGenerationDetail`, then Web navigates to `/signal/:id/transmitting`.
- Good: transmitting page polls `GET /api/signals/:id`; `COMPLETED` redirects
  to `/result/:id`.
- Good: low/medium-risk completion renders a private owner preview; the public
  result, galaxy, random, match, save, resonance, and share-card surfaces stay
  empty until owner publish.
- Good: owner publish moves the same row to `PUBLISHED` and returns the public
  detail response.
- Good: high-risk seed completes as `REVIEWING`; result page shows the owner
  review state.
- Base: matching store is unavailable; generation still completes with no
  matches.
- Base: AI generation fails; owner stays on the transmitting page and can retry
  by creating a new signal with the same seed.
- Bad: homepage submit waits for full AI generation before navigation.
- Bad: pending placeholder rows enter galaxy, random, match, save, resonance,
  or share-card surfaces.
- Bad: private preview rows enter galaxy, random, match, save, resonance, or
  share-card surfaces before publish.
- Bad: matching failure aborts signal creation.

### 6. Tests Required

- Service test: normal completion writes `generationStatus="COMPLETED"` and
  final `status="DRAFT"`.
- Service test: high-risk completion writes `generationStatus="COMPLETED"` and
  final `status="REVIEWING"`.
- Service or route test: matching lookup failure still completes generation and
  returns `matches: []` after completion.
- Service or route test: embedding write is queued once after completion.
- Route test: `POST /api/signals` returns `SignalGenerationDetail` with
  `generationStatus="RECEIVED"` and `status="DRAFT"`.
- Route test: owner `GET /api/signals/:id` for generating rows returns
  `SignalGenerationDetail` without `expandedText`.
- Route test: non-owner `GET /api/signals/:id` for generating rows returns
  `404 NOT_FOUND`.
- Route test: owner `GET /api/signals/:id` for a completed private preview
  returns `SignalPreviewDetail`.
- Route test: non-owner and anonymous `GET /api/signals/:id` for a completed
  private preview return `404 NOT_FOUND`.
- Route test: owner `POST /api/signals/:id/publish` returns `SignalDetail` with
  `status="PUBLISHED"`.
- Route test: non-owner publish returns `404 NOT_FOUND`; incomplete or
  reviewing publish returns `409 CONFLICT`.
- Route test: `POST /api/signals/generate` returns `404 NOT_FOUND`.

### 7. Wrong vs Correct

#### Wrong

```typescript
const signal = await createPublishedSignal(request.frequency, body, request.log);
return sendData(reply, request, { signal }, 201);
```

#### Correct

```typescript
const signal = await createPendingSignal(request.frequency, body, request.log);
return sendData(reply, request, { signal }, 201);
```

## Scenario: Reviewing Signal Result Visibility

### 1. Scope / Trigger

- Trigger: changing `GET /api/signals/:id`, result-page visibility, signal
  moderation status rendering, or shared signal response DTOs.
- Applies to backend API, Web, and future Android clients because reviewing
  state is exposed through the shared JSON API.

### 2. Signatures

```text
GET /api/signals/:id

PUBLISHED -> 200 { signal: SignalDetail }
REVIEWING + owner -> 200 { signal: SignalDetail }
REVIEWING + non-owner/anonymous -> 200 { signal: ReviewingSignalRestrictedDetail }
DRAFT + owner -> 200 { signal: SignalPreviewDetail }
DRAFT + non-owner/anonymous -> 404 NOT_FOUND
DELETED/HIDDEN/unknown -> 404 NOT_FOUND

ReviewingSignalRestrictedDetail:
  id: uuid
  status: REVIEWING
  visibility: REVIEW_RESTRICTED
  createdAt: ISO datetime
  updatedAt: ISO datetime
```

### 3. Contracts

- Published signals keep the full `SignalDetail` response and may include
  matches plus viewer save state.
- Reviewing owner access is based on `frequencyId` or `authorId` matching the
  authenticated frequency id.
- Reviewing owner responses confirm the saved seed and status. They must not
  compute same-frequency matches or enable interactions in client UI.
- Reviewing non-owner and anonymous responses must use
  `ReviewingSignalRestrictedDetail`; they must not include `seedText`,
  `expandedText`, `rawInput`, `fingerprint`, `program`, matches, audio, save,
  resonance, or share-card content.
- Web result pages must forward the incoming cookie when server-rendering
  `/result/:id`; otherwise the owner would be treated as anonymous.

### 4. Validation & Error Matrix

- Invalid UUID -> `400 VALIDATION_ERROR`.
- Missing id -> `404 NOT_FOUND`.
- `PUBLISHED` -> full detail.
- `DRAFT` owner -> private preview detail with `visibility="PREVIEW_OWNER"`.
- `DRAFT` non-owner or anonymous -> `404 NOT_FOUND`.
- `REVIEWING` owner -> full detail with `status="REVIEWING"` and empty matches.
- `REVIEWING` non-owner or anonymous -> restricted detail only.
- `HIDDEN` or `DELETED` -> `404 NOT_FOUND`.

### 5. Good/Base/Bad Cases

- Good: author opens `/result/:id` for a reviewing signal and sees only the
  original seed plus review status.
- Good: another frequency opens the same URL and sees
  `visibility="REVIEW_RESTRICTED"` without seed or expanded text.
- Base: published signal behavior stays unchanged.
- Bad: returning full `SignalDetail` for non-owner reviewing access and hiding
  it only in Web UI.
- Bad: treating reviewing access as `404`, which prevents clients from showing
  a stable "temporarily not public" state.

### 6. Tests Required

- Route test: owner `GET /api/signals/:id` for `REVIEWING` returns `200`,
  `status="REVIEWING"`, and `seedText`.
- Route test: non-owner `GET /api/signals/:id` for `REVIEWING` returns `200`,
  `visibility="REVIEW_RESTRICTED"`, and no `seedText` or `expandedText`.
- Route test: anonymous reviewing access also returns restricted detail.
- Web typecheck must prove result page branches before reading full-detail
  fields.

### 7. Wrong vs Correct

#### Wrong

```typescript
const detail = await getSignalDetail(id);
return sendData(reply, request, { signal: detail });
```

#### Correct

```typescript
const signal = await getSignalDetailForFrequency(id, request.frequency);
return sendData(reply, request, { signal });
```

## Scenario: Shared API Contract Package

### 1. Scope / Trigger

- Trigger: adding, removing, renaming, or changing any public JSON DTO, API envelope, current product enum, admin DTO, auth DTO, signal DTO, recap DTO, or theme radio DTO used by both `apps/api` and `apps/web`.
- Applies to `packages/contracts`, `apps/api`, and `apps/web`.
- Does not apply to database row types, repository records, service-only inputs, or Zod validation schemas that are not public response contracts.

### 2. Signatures

Workspace package:

```text
@unfreq/contracts
@unfreq/contracts/api
@unfreq/contracts/auth
@unfreq/contracts/signals
@unfreq/contracts/admin
@unfreq/contracts/theme-radio
```

Root commands:

```text
npm run contracts:typecheck
npm run contracts:build
npm run check
```

### 3. Contracts

- Shared public DTOs live under `packages/contracts/src/*.ts`.
- `apps/api` public response types must import from or re-export from `@unfreq/contracts`.
- `apps/web` must import API/domain DTOs from `@unfreq/contracts` or one of its subpaths, not from hand-copied app-local type files.
- `apps/web/src/types/core.ts` may contain frontend-only view types, such as navigation labels. It must not redefine API DTOs.
- Backend internal record types with `Date`, nullable database fields, storage keys, or raw JSON columns stay in `apps/api`.
- API runtime validation remains in `apps/api` Zod schemas. The contracts package is the TypeScript contract source, not a replacement for request/response validation.
- Package runtime exports point at `packages/contracts/dist/*`. Commands that run API/Web build, dev, start, or e2e must build contracts first.

### 4. Validation & Error Matrix

- Contract type error -> `npm run contracts:typecheck` exits non-zero.
- API or Web imports a missing contract export -> `npm run api:typecheck` or `npm run web:typecheck` exits non-zero.
- Runtime import is used before contracts are built -> module resolution can fail from `dist`; use root scripts that run `contracts:build`.
- Response shape mismatch despite matching TypeScript types -> backend Zod schema or route test must catch it.

### 5. Good/Base/Bad Cases

- Good: `apps/web` imports `SignalDetail` from `@unfreq/contracts`.
- Good: `apps/api/src/modules/signals/types.ts` re-exports public `SignalDetail` while keeping `SignalRecord` local.
- Base: a backend-only repository type stays app-local because it includes `Date` and nullable database fields.
- Bad: redefining `GalaxyResponse` in `apps/web/src/types/core.ts`.
- Bad: importing Drizzle table types or repository records into `packages/contracts`.

### 6. Tests Required

- Run `npm run contracts:typecheck` after editing `packages/contracts`.
- Run `npm run check` for cross-layer contract changes.
- Search for app-local duplicate DTOs after migration:

```bash
rg "export type (FrequencyIdentity|SignalDetail|GalaxyResponse|ApiResponse)" apps/api/src apps/web/src packages/contracts/src
```

Expected result: public DTO definitions live in `packages/contracts`; app-local hits are internal records, wrappers, or frontend-only view types.

### 7. Wrong vs Correct

#### Wrong

```typescript
export type SignalDetail = {
  id: string;
  expandedText: string;
};
```

#### Correct

```typescript
import type { SignalDetail } from "@unfreq/contracts";
```

## Scenario: Retired Legacy Product API Boundary

### 1. Scope / Trigger

- Trigger: route registration, report target handling, admin moderation, AI review targets, or any work touching old voice, bottle, channel, translate, radio-report, or legacy AI product surfaces.
- Applies to Web and future clients because the default Fastify app is the public product boundary.
- This scenario does not require database table or migration cleanup. Historical tables/enums may remain until a separate data-retirement task.

### 2. Signatures

Default registered API routes must not include:

```text
GET/POST /api/channels/*
GET/POST /api/voices/*
GET/POST /api/bottles/*
GET/POST /api/translate/*
GET/POST /api/radio-report/*
GET/POST /api/ai/*
```

Current report and AI review target types:

```text
ReportTargetType: USER | SIGNAL
AdminAiReviewTargetType: USER | SIGNAL
```

### 3. Contracts

- `buildApp()` registers only current product routes by default: auth, frequencies, `/api/me`, frontend diagnostics, reports, signals, monthly recaps, theme radio, images, and admin.
- Retired product API modules must not be reintroduced as default registrations.
- Reports and AI reviews must reject retired target types such as `VOICE`, `BOTTLE`, `BOTTLE_ECHO`, or old `ECHO`.
- Admin overview and moderation must use signals, frequencies, reports, AI reviews, and admin actions. It must not compute primary metrics from old voices or bottles.
- Theme radio remains current as a light galaxy theme hint; it is not the retired auto-playing radio-report product.

### 4. Validation & Error Matrix

- Request to retired API path -> `404 NOT_FOUND` from route absence.
- Report with retired `targetType` -> `400 VALIDATION_ERROR`.
- Admin AI review with retired target type -> rejected by schema or ignored by typed query boundary.
- Historical rows in retired database tables -> no public API exposure unless a future task restores a route.

### 5. Good/Base/Bad Cases

- Good: `/api/signals/:id/resonance` and `/api/signals/:id/save` continue to work after voice/bottle APIs are removed.
- Good: `/api/reports` accepts `targetType=SIGNAL` or `USER` and rejects `VOICE`.
- Base: old database enums and tables still exist but are unreachable through default API routes.
- Bad: keeping `/api/voices` registered only to return a redirect-style compatibility response.
- Bad: adding a Web-only replacement for a retired API instead of using a current JSON API.

### 6. Tests Required

- Root `npm run check` after removing or restoring any default route.
- Route test covering rejected legacy report target types.
- Admin route/type tests proving overview and moderation DTOs use current signal/frequency/report targets.
- Residual search for retired path names across app code, with allowed hits limited to documentation, negative tests, and env keys unrelated to retired products.

### 7. Wrong vs Correct

#### Wrong

```typescript
await app.register(voiceRoutes, { prefix: "/api/voices" });
```

#### Correct

```typescript
await app.register(signalRoutes, { prefix: "/api/signals" });
```

## Scenario: Account-Required Frequency Sessions

### 1. Scope / Trigger

- Trigger: authentication, homepage current-frequency loading, or any change touching `GET /api/me`, `POST /api/frequencies`, guest login, or session restoration.
- Applies to Web and future clients because the login boundary lives in the shared Fastify API.
- The current product requires explicit account registration or login before user-owned writes.

### 2. Signatures

API:

```text
GET  /api/me
POST /api/auth/register
POST /api/auth/frequency/login
POST /api/auth/guest
POST /api/frequencies
```

Session:

```text
Cookie: unfreq_session=<signed-session-token>
```

Database field:

```text
users.is_guest boolean
```

### 3. Contracts

- `POST /api/auth/register` creates a non-guest password frequency with a random `FM XX.XX` and sets `unfreq_session`.
- `POST /api/auth/frequency/login` restores only password-backed FM accounts.
- `POST /api/auth/guest` must stay disabled.
- `POST /api/frequencies` is now a session restore endpoint only; it must not create a new frequency without a valid non-guest session.
- `GET /api/me` must return a frequency only for a valid non-guest session.
- Legacy session tokens pointing at `users.is_guest = true` must be treated as unauthenticated.

### 4. Validation & Error Matrix

- No session token -> `401 UNAUTHORIZED`.
- Invalid, expired, banned, or guest session token -> `401 UNAUTHORIZED`.
- `POST /api/auth/guest` -> `403 FORBIDDEN`.
- `POST /api/frequencies` without a non-guest session -> `401 UNAUTHORIZED`.

### 5. Good/Base/Bad Cases

- Good: a browser with an old guest cookie opens `/`; `/api/me` returns `401`, and Web shows login/register entry.
- Base: a registered FM opens `/`; `/api/me` returns the current FM.
- Bad: old guest cookies silently restore `FM XX.XX` and let the homepage look logged in.

### 6. Tests Required

- Auth route test: old guest session cookie returns `401` from `/api/me`.
- Auth route test: `/api/auth/guest` returns `403` and creates no user.
- Auth route test: `/api/frequencies` without a session returns `401` and creates no user.
- Frontend typecheck/lint must cover current-frequency helpers that hide `isGuest` identities.

### 7. Wrong vs Correct

#### Wrong

```typescript
if (user === undefined || user.status === "BANNED") {
  return undefined;
}
```

#### Correct

```typescript
if (user === undefined || user.status === "BANNED" || user.isGuest) {
  return undefined;
}
```

## Scenario: Unfreq Resonance Signal API Contracts

### 1. Scope / Trigger

- Trigger: homepage seed emission, result page, same-frequency matching, on-demand TTS, resonance, return-visit echo summary, share cards, or `/galaxy` changes.
- Applies to Web and the future Android Kotlin client.
- This is the current product contract and supersedes legacy signal program, expression translation, and risk radar contracts for new core work.

### 2. Signatures

API:

```text
POST /api/frequencies
GET  /api/me
GET  /api/me/signals
GET  /api/me/echo-summary
POST /api/me/echo-summary/seen
GET  /api/me/saves
GET  /api/me/recaps/latest
POST /api/me/recaps
GET  /api/me/recaps/:yearMonth

POST /api/signals
GET  /api/signals/:id
GET  /api/signals/:id/matches
GET  /api/signals/galaxy?band=<signal_primary_band>
GET  /api/signals/random
POST /api/signals/:id/resonate
POST /api/signals/:id/resonance
DELETE /api/signals/:id/resonance
POST /api/signals/:id/save
DELETE /api/signals/:id/save

GET  /api/signals/:id/audio
POST /api/signals/:id/audio
GET  /api/og/:signalId

GET  /api/theme-radio/current
GET  /api/admin/theme-radio-days
POST /api/admin/theme-radio-days
```

Database:

```text
frequencies(
  id,
  code,
  device_hash nullable,
  frequency_name nullable,
  password_hash nullable,
  is_guest,
  last_seen_at nullable,
  created_at,
  updated_at
)

signals(
  id,
  frequency_id,
  seed_text,
  expanded_text,
  embedding vector nullable,
  embedding_provider nullable,
  embedding_model nullable,
  embedding_dimensions nullable,
  embedding_version nullable,
  galaxy_position jsonb nullable,
  primary_band signal_primary_band nullable,
  sub_feeling signal_sub_feeling nullable,
  emotion_tone text nullable,
  core_need text nullable,
  inner_conflict text nullable,
  resonance_keywords jsonb nullable,
  follow_up_question nullable,
  parent_signal_id nullable,
  audio_status,
  audio_error nullable,
  audio_storage_key nullable,
  audio_url nullable,
  resonance_count,
  source,
  status,
  created_at,
  updated_at
)

signal_resonances(
  id,
  signal_id,
  frequency_id nullable,
  actor_key,
  created_at
)

signal_saves(
  id,
  signal_id,
  frequency_id,
  created_at
)

monthly_recaps(
  id,
  frequency_id,
  year_month,
  signal_count,
  total_resonance_count,
  top_signal_id nullable,
  theme_words,
  summary,
  generated_at,
  created_at,
  updated_at
)

theme_radio_days(
  id,
  theme,
  title,
  description nullable,
  starts_at,
  ends_at nullable,
  is_active,
  created_at,
  updated_at
)
```

Signal frequency enums:

```text
signal_primary_band:
  self_mirror | emotional_undercurrent | relationship_nebula | time_ring |
  meaning_deep_space | stagnant_orbit | era_cluster

signal_sub_feeling:
  self_doubt | identity_confusion | self_acceptance |
  fatigue | anxiety | numbness |
  loneliness | need_to_be_understood | relationship_disappointment |
  growth_anxiety | nostalgia | future_uncertainty |
  effort_doubt | goal_loss | existential_questioning |
  procrastination | starting_difficulty | fear_of_failure |
  efficiency_anxiety | work_alienation | modern_loneliness
```

### 3. Contracts

`POST /api/frequencies` response:

```json
{
  "data": {
    "frequency": {
      "id": "uuid",
      "code": "FM 78.23",
      "isGuest": true,
      "role": "USER",
      "status": "ACTIVE",
      "lastSeenAt": null,
      "createdAt": "2026-05-15T00:00:00.000Z"
    }
  },
  "meta": { "requestId": "req_123" }
}
```

`code` must display as `FM XX.XX`, with both numeric groups zero-padded.

During the legacy auth migration, backend responses may also include `freqCode` as a compatibility alias for older web surfaces. New core clients must read `code` first and must not require `sessionToken` in the JSON body; session continuity is carried by the HttpOnly cookie set by the API.

`POST /api/signals` request:

```json
{
  "seedText": "所有忙，都是不敢停下来面对什么都没做。",
  "parentSignalId": "optional-uuid"
}
```

- `seedText`: trimmed string, 5-200 chars.
- `parentSignalId`: optional, must point to a visible signal for the current frequency when supplied.

`POST /api/signals` response:

```json
{
  "data": {
    "signal": {
      "id": "uuid",
      "frequencyCode": "FM 78.23",
      "seedText": "所有忙，都是不敢停下来面对什么都没做。",
      "expandedText": "凌晨两点。第十一杯咖啡。第七版方案。我们以为自己在工作，其实是在用键盘声盖住房间里的那个问题：这一年到底做出了什么。忙是当代最体面的麻醉剂，它让你显得勤奋，也让你不用承认，很多事不是没时间做，是不敢面对。",
      "followUpQuestion": "你最近一次真正停下来，是什么时候？",
      "parentSignalId": null,
      "matches": [],
      "audio": {
        "signalId": "uuid",
        "status": "NOT_REQUESTED"
      },
      "resonanceCount": 0,
      "frequencyProfile": {
        "primaryBand": "meaning_deep_space",
        "subFeeling": "effort_doubt",
        "displayText": "意义深空 · 努力怀疑",
        "emotionTone": "清醒疲惫",
        "coreNeed": "确认努力是否值得",
        "innerConflict": "想继续努力，但怀疑努力是否真的通向自己想要的生活",
        "resonanceKeywords": ["忙碌", "空转", "自我价值"]
      },
      "hasResonated": false,
      "source": "human",
      "status": "PUBLISHED",
      "createdAt": "2026-05-15T00:00:00.000Z",
      "updatedAt": "2026-05-15T00:00:00.000Z"
    }
  },
  "meta": { "requestId": "req_123" }
}
```

AI expansion rules:

- `expandedText` is an 80-180 Chinese-character concrete scene when provider output is available.
- It preserves the seed direction, adds concrete objects, visible action, and sensory detail.
- It must not become relationship advice, therapy, diagnosis, or a risk-scoring explanation.
- It must not mention signal, frequency, resonance, FM, radio, galaxy, star map, strangers, being seen, being understood, healing, or the universe.
- Expansion input is only `seedText`, structured seed classification, and optional rewrite instruction. It must not include fingerprint, frequency profile, risk labels, or product-world metadata.
- Provider output must be parsed and validated before storage.
- Signal fingerprint output must normalize to one fixed `primaryBand` and one
  `subFeeling`. If the provider returns a valid enum pair that does not belong
  together, the service must keep the primary band and choose the closest allowed
  sub-feeling under that band.
- Signal detail responses must return a complete current `fingerprint` DTO and
  `frequencyProfile` even when stored `signals.fingerprint_json` is legacy or
  partial. Normalize the response from database columns plus
  `normalizeSignalFrequencyProfile`; do not pass raw `fingerprint_json` directly
  through the route response schema.
- `resonanceKeywords` are 1-3 semantic hints only. They are never treated as a
  formal tag taxonomy.
- Provider failure returns a deterministic fallback in the same DTO shape.

`GET /api/signals/:id/matches` response:

```json
{
  "data": {
    "matches": [
      {
        "id": "uuid",
        "frequencyCode": "FM 21.04",
        "seedText": "optional seed preview",
        "expandedText": "我以前以为重要是一种状态，后来发现它常常只是一种表演。",
        "resonanceCount": 12,
        "distance": 0.18,
        "source": "PUBLISHED",
        "frequencyProfile": {
          "primaryBand": "meaning_deep_space",
          "subFeeling": "effort_doubt",
          "displayText": "意义深空 · 努力怀疑",
          "emotionTone": "清醒疲惫",
          "coreNeed": "确认努力是否值得",
          "innerConflict": "想继续努力，但怀疑努力是否真的通向自己想要的生活",
          "resonanceKeywords": ["忙碌", "空转", "自我价值"]
        }
      }
    ]
  },
  "meta": { "requestId": "req_123" }
}
```

`source` is `PUBLISHED`, `SEEDED`, or `FALLBACK` for match DTOs. Seeded matches may be shown, but must not be stored as if they were human signals.

`GET /api/signals/galaxy` response:

```json
{
  "data": {
    "points": [
      {
        "id": "uuid",
        "frequencyCode": "FM 78.23",
        "expandedText": "expanded text preview",
        "resonanceCount": 7,
        "source": "human",
        "theme": "意义深空 · 努力怀疑",
        "frequencyProfile": {
          "primaryBand": "meaning_deep_space",
          "subFeeling": "effort_doubt",
          "displayText": "意义深空 · 努力怀疑",
          "emotionTone": "清醒疲惫",
          "coreNeed": "确认努力是否值得",
          "innerConflict": "想继续努力，但怀疑努力是否真的通向自己想要的生活",
          "resonanceKeywords": ["忙碌", "空转", "自我价值"]
        },
        "isMine": true,
        "position": { "x": 0.12, "y": -0.44, "z": 0.31 },
        "matchIds": ["uuid-2", "uuid-3"],
        "createdAt": "2026-05-15T00:00:00.000Z"
      }
    ],
    "bandOptions": [
      { "label": "自我镜像", "value": "self_mirror" },
      { "label": "情绪暗流", "value": "emotional_undercurrent" },
      { "label": "关系星云", "value": "relationship_nebula" },
      { "label": "时间星环", "value": "time_ring" },
      { "label": "意义深空", "value": "meaning_deep_space" },
      { "label": "停滞轨道", "value": "stagnant_orbit" },
      { "label": "时代星群", "value": "era_cluster" }
    ],
    "generatedAt": "2026-05-15T00:00:00.000Z",
    "themeOptions": ["意义深空", "关系星云"]
  },
  "meta": { "requestId": "req_123" }
}
```

Coordinates must be finite numbers. API may return an empty `points` array; frontend must render a truthful empty state.
`matchIds` should reference published signal ids already present in the galaxy response when available. The galaxy route must not call external AI providers just to draw links.
`theme` remains as a compatibility display field and should equal
`frequencyProfile.displayText` for new signals. New clients must use
`frequencyProfile`.
`GET /api/signals/galaxy?band=<signal_primary_band>` filters public points by
the fixed primary band. Legacy `?theme=` may remain as a compatibility alias,
but new UI must not expose theme-tag filtering.
`bandOptions` always returns the fixed seven primary bands. `themeOptions`
remains only for older clients and must not drive new Web UI.

`POST /api/signals/:id/resonance`, `POST /api/signals/:id/resonate`, and `DELETE /api/signals/:id/resonance` response:

```json
{
  "data": {
    "resonance": {
      "hasResonated": true,
      "resonanceCount": 13
    }
  },
  "meta": { "requestId": "req_123" }
}
```

The pair `(signal_id, actor_key)` is unique. Repeated resonance is idempotent and must not increment count twice.
`POST /api/signals/:id/resonate` is a compatibility alias for the same create-resonance behavior as `POST /api/signals/:id/resonance`.
`DELETE` is also idempotent and returns the current count with `hasResonated=false`.

`POST /api/signals/:id/save` and `DELETE /api/signals/:id/save` response:

```json
{
  "data": {
    "save": {
      "hasSaved": true
    }
  },
  "meta": { "requestId": "req_123" }
}
```

Saving requires a frequency session. It only works for `PUBLISHED` signals and never notifies the signal author.
The pair `(frequency_id, signal_id)` is unique. Repeated save is idempotent.
`DELETE` is idempotent and returns `hasSaved=false`.
`GET /api/signals/:id` may include `hasSaved` when the request has a frequency session.

`GET /api/me/signals` response:

```json
{
  "data": {
    "signals": [
      {
        "id": "uuid",
        "seedText": "所有忙，都是不敢停下来面对什么都没做。",
        "expandedText": "凌晨两点。第十一杯咖啡。第七版方案。我们以为自己在工作，其实是在用键盘声盖住房间里的那个问题：这一年到底做出了什么。",
        "parentSignalId": null,
        "resonanceCount": 12,
        "status": "PUBLISHED",
        "createdAt": "2026-05-15T00:00:00.000Z"
      }
    ],
    "totalResonanceCount": 12
  },
  "meta": { "requestId": "req_123" }
}
```

The list contains only the current frequency's own signals and excludes `DELETED`.

`GET /api/me/saves` response:

```json
{
  "data": {
    "signals": [
      {
        "id": "uuid",
        "seedText": "所有忙，都是不敢停下来面对什么都没做。",
        "expandedText": "凌晨两点。第十一杯咖啡。第七版方案。",
        "parentSignalId": null,
        "resonanceCount": 12,
        "status": "PUBLISHED",
        "theme": "孤独",
        "authorFreqCode": "FM 78.23",
        "savedAt": "2026-05-15T00:00:00.000Z",
        "createdAt": "2026-05-14T00:00:00.000Z"
      }
    ]
  },
  "meta": { "requestId": "req_123" }
}
```

The list is private to the current frequency, ordered by save time, and filters out non-`PUBLISHED` signals.

`GET /api/me/recaps/latest`, `POST /api/me/recaps`, and `GET /api/me/recaps/:yearMonth` use this DTO:

```json
{
  "data": {
    "recap": {
      "id": "uuid",
      "frequencyId": "uuid",
      "yearMonth": "2026-04",
      "signalCount": 4,
      "totalResonanceCount": 9,
      "summary": "这个月的信号靠近孤独和等待，也留下了继续往前看的余地。",
      "themeWords": ["孤独", "等待"],
      "topSignal": {
        "id": "uuid",
        "seedText": "所有忙，都是不敢停下来面对什么都没做。",
        "expandedText": "凌晨两点。第十一杯咖啡。",
        "resonanceCount": 5,
        "status": "PUBLISHED",
        "createdAt": "2026-04-10T00:00:00.000Z"
      },
      "generatedAt": "2026-05-15T00:00:00.000Z",
      "createdAt": "2026-05-15T00:00:00.000Z",
      "updatedAt": "2026-05-15T00:00:00.000Z"
    }
  },
  "meta": { "requestId": "req_123" }
}
```

`POST /api/me/recaps` accepts `{ "yearMonth": "YYYY-MM" }`, optional. If omitted, it generates the previous UTC month.
Generation requires at least 3 `PUBLISHED` signals owned by the current frequency in that month and upserts by `(frequency_id, year_month)`.
The AI recap returns 1-3 theme words and a summary under 100 chars. Provider failure must use a deterministic fallback without medical, psychological, personality, or exaggerated claims.
`GET /api/me/recaps/latest` returns `{}` when none exists. `GET /api/me/recaps/:yearMonth` returns `404` when missing.

`GET /api/theme-radio/current` and admin theme-radio endpoints use this DTO:

```json
{
  "data": {
    "themeRadioDay": {
      "id": "uuid",
      "theme": "孤独",
      "title": "低频夜行日",
      "description": "这一期只做轻提示和星图筛选。",
      "startsAt": "2026-05-15T00:00:00.000Z",
      "endsAt": "2026-05-18T00:00:00.000Z",
      "isActive": true,
      "createdAt": "2026-05-15T00:00:00.000Z",
      "updatedAt": "2026-05-15T00:00:00.000Z"
    }
  },
  "meta": { "requestId": "req_123" }
}
```

Current theme selection filters `is_active=true`, `starts_at <= now`, and `ends_at IS NULL OR ends_at > now`, ordered by newest `starts_at`.
Only admins can list or create theme radio days. A theme radio day only affects homepage hints and `/galaxy` filtering; it must not create rankings or author notifications.
`parentSignalId` is nullable and lets Web and Android distinguish ordinary signals from continued signals and build a local thought tree. The parent may be missing from the same response when it is outside the current page limit or later hidden/deleted; clients must render the child without assuming the parent object is present.
During the legacy `users` auth migration, this query is keyed by `signals.author_id`.
Authors may see their own `DRAFT`, `REVIEWING`, and `HIDDEN` signals on `/me`, but public result and galaxy routes must still filter to `PUBLISHED`.

`GET /api/me/echo-summary` response:

```json
{
  "data": {
    "summary": {
      "frequencyId": "uuid",
      "newResonanceCount": 5,
      "totalResonanceCount": 47,
      "since": "2026-05-14T00:00:00.000Z",
      "topSignal": null
    }
  },
  "meta": { "requestId": "req_123" }
}
```

If the backend has no persisted `lastSeenAt` for the current frequency, `since` is omitted and `newResonanceCount` must be `0`; do not reuse `totalResonanceCount` as a fake new-count. Web may compare `totalResonanceCount` with a browser-local previously seen total to render a return prompt, but the API remains the source of truth for resonance totals.

`POST /api/me/echo-summary/seen` returns the same DTO shape and advances the current frequency's `last_seen_at` watermark after the summary has been seen or dismissed. It must not create fake counts; a later `GET /api/me/echo-summary` should only count resonances created after the updated watermark.

Signal audio status:

```text
NOT_REQUESTED -> no TTS work has started
PENDING       -> user clicked playback and service-side TTS is generating
READY         -> audioUrl is present
FAILED        -> text remains usable; playback action shows failure
```

`GET /api/signals/:id/audio` is read-only. It returns current audio status and must not start TTS generation.

`POST /api/signals/:id/audio` starts or reuses TTS for `expandedText`. It must not use `seedText` unless an explicit fallback reason is stored. Audio is never required for signal creation.

Embedding and reranker environment:

```text
EMBEDDING_PROVIDER=dashscope
EMBEDDING_BASE_URL=
EMBEDDING_DIMENSIONS=1024
EMBEDDING_MODEL=text-embedding-v3
EMBEDDING_API_KEY=
EMBEDDING_VERSION=v1
RERANKER_PROVIDER=disabled
RERANKER_BASE_URL=
RERANKER_MODEL=local-disabled
RERANKER_API_KEY=
```

`generateEmbedding(text)` returns `{ embedding, provider, model, dimensions, version }`.
Embedding writes persist all metadata fields with the vector. Same-frequency semantic recall filters by `embedding_version`, then reranker scores candidates when configured. Reranker disabled or failed means the service keeps embedding distance ordering.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Invalid seed length | 400 | VALIDATION_ERROR |
| Invalid parent signal | 404 | NOT_FOUND |
| Invalid galaxy band query | 400 | VALIDATION_ERROR |
| AI provider unavailable during expansion | 201 | fallback expanded text |
| AI fingerprint returns mismatched band/sub-feeling | 201 | normalized fixed frequency profile |
| Legacy/partial stored fingerprint JSON | 200 | complete current fingerprint and frequency profile |
| Embedding provider unavailable | 201 | signal created, keyword/seed fallback for matches |
| Reranker provider unavailable | 200/201 | semantic candidates ranked by embedding distance |
| Hidden/deleted signal detail | 404 | NOT_FOUND |
| Duplicate resonance | 200 | idempotent current count |
| Duplicate resonance delete | 200 | idempotent current count |
| Missing actor for anonymous resonance | 400 | VALIDATION_ERROR |
| Echo summary seen without session | 401 | UNAUTHORIZED |
| Audio requested for hidden/deleted signal | 404 | NOT_FOUND |
| TTS provider unavailable | 200 | audio.status `FAILED` |
| Share image for hidden/deleted signal | 404 | NOT_FOUND |

### 5. Good/Base/Bad Cases

- Good: create signal writes seed and expanded text first, then best-effort embedding and galaxy position.
- Good: TTS starts only after a playback request.
- Good: matching uses `expanded_text` embedding, stores embedding metadata, isolates recall by `embedding_version`, then deterministic fallback fills missing results.
- Good: matching blends embedding similarity with fixed frequency profile signals.
- Good: reranker failure keeps the response usable through embedding distance ordering.
- Good: seed content is marked `source='seed'` and never represented as human content.
- Good: public detail and galaxy queries filter `PUBLISHED` in SQL.
- Good: galaxy filters by fixed `primary_band`, colors by the seven primary bands, and keeps sub-feelings only in point detail.
- Good: random tuning filters out the current frequency's own published signals when another published signal is available.
- Good: return-visit echo summary reads `frequencies.last_seen_at`, counts only current-frequency signals, and advances the watermark only through the seen endpoint.
- Good: galaxy route derives coordinates/links from already fetched rows and never calls embedding or reranker providers.
- Base: local demo with no AI keys still creates deterministic expanded text and seeded matches.
- Bad: call an AI provider from Web.
- Bad: require audio or embedding success before returning a created signal.
- Bad: expose `DRAFT`, `HIDDEN`, or `DELETED` signals through public result or galaxy APIs.
- Bad: random tuning always returns the current frequency's own signal when public stranger signals exist.
- Bad: treat total resonance count as new resonance count when no `last_seen_at` exists.
- Bad: add comments, follows, private messages, hot rankings, or feed-style community APIs to the core loop.
- Bad: return risk radar, relationship advice, or translation DTOs from `POST /api/signals`.
- Bad: show or match through an unbounded tag wall in core signal surfaces.

### 6. Tests Required

- Route test: create signal success returns full DTO and shared envelope.
- Route test: invalid seed returns `400 VALIDATION_ERROR`.
- Service test: AI invalid output returns fallback expanded text and follow-up question.
- Service test: embedding failure keeps signal creation successful.
- Service test: same-frequency matching returns stable Top3 or fallback.
- AI service test: mismatched `primaryBand` / `subFeeling` provider output is normalized before storage.
- Service test: embedding metadata is written with the vector.
- Service test: reranker disabled/failure falls back to embedding distance.
- Route test: detail and galaxy only expose published signals.
- Route test: detail returns `200` and a complete current fingerprint when
  stored `fingerprint_json` is legacy or partial.
- Route test: filtered galaxy responses accept `band`, return `bandOptions`, and keep only matching public points.
- Route test: repeated resonance is idempotent.
- Route test: `/api/signals/:id/resonate` aliases create-resonance behavior.
- Route test: repeated resonance delete is idempotent.
- Route test: echo summary counts only the current frequency's signals.
- Route test: echo summary seen advances `last_seen_at` and later summary calls do not repeat the same batch.
- Route test: audio starts on explicit request and handles provider failure.
- Route test: share image rejects hidden/deleted signals.

### 7. Wrong vs Correct

#### Wrong

```typescript
const audio = await createTtsAudio(seedText);
const embedding = await generateEmbedding(seedText);
return createSignal({ seedText, audio, embedding });
```

#### Correct

```typescript
const expanded = await expandSeedOrFallback(seedText);
const signal = await createPublishedSignal({ seedText, expandedText: expanded.text });
queueEmbeddingWrite(signal.id, expanded.text, embeddingMeta);
return toSignalDetailDto(signal);
```

#### Wrong

```typescript
return {
  fingerprint: signal.fingerprintJson,
  frequencyProfile: getSignalFrequencyProfile(signal),
};
```

#### Correct

```typescript
const frequencyProfile = getSignalFrequencyProfile(signal);

return {
  fingerprint: getSignalResponseFingerprint(signal),
  frequencyProfile,
};
```

#### Wrong

```typescript
app.get("/signals/galaxy", async () => db.select().from(signals));
```

#### Correct

```typescript
return signalRepository.listPublishedGalaxyPoints({ frequencyId });
```

## Scenario: Admin Provider Config API

### 1. Scope / Trigger

- Trigger: admin AI provider settings, provider failover, encrypted API key storage, or runtime AI provider changes.
- Applies to Web admin, Fastify admin routes, AI runtime modules, and deployment env wiring.
- This is a cross-layer and secret-handling contract because the UI writes configuration into PostgreSQL while runtime AI calls read it immediately.

### 2. Signatures

API:

```text
GET /api/admin/provider-configs
PUT /api/admin/provider-configs/:kind
POST /api/admin/provider-configs/:kind
PUT /api/admin/provider-configs/:kind/:id
DELETE /api/admin/provider-configs/:kind/:id
```

Path params:

```text
kind: chat | embedding | tts | reranker
id: provider_configs.id uuid
```

Create/update body:

```json
{
  "provider": "dashscope",
  "model": "text-embedding-v3",
  "baseUrl": "https://dashscope.aliyuncs.com/compatible-mode/v1",
  "apiKey": "optional-new-key",
  "clearApiKey": false,
  "isEnabled": true,
  "priority": 20,
  "extraJson": {
    "dimensions": 1024,
    "version": "v1"
  }
}
```

Response DTO:

```text
AdminProviderConfigView:
  id?: uuid
  kind: chat | embedding | tts | reranker
  provider: string
  model: string
  baseUrl: string
  isEnabled: boolean
  priority: number
  source: database | environment
  status: configured | degraded | disabled
  keyState: configured | missing
  apiKeyLast4?: string
  extraJson: Record<string, string | number | boolean | null>
  updatedAt?: ISO datetime
  updatedBy?: uuid
```

### 3. Contracts

- Admin access is required for all endpoints.
- `GET /api/admin/provider-configs` returns database rows per kind plus one environment fallback row for each kind.
- Database rows sort by ascending `priority`; environment rows use `priority=1000` and have no `id`.
- The `priority=1000` value on environment rows is display/sort only. Clients must not send it back in create/update bodies. Writable database priorities remain `1..999`.
- `POST /api/admin/provider-configs/:kind` creates a new database backup row.
- `PUT /api/admin/provider-configs/:kind/:id` updates only the row matching both `kind` and `id`.
- `DELETE /api/admin/provider-configs/:kind/:id` deletes only the row matching both `kind` and `id` and returns a sanitized DTO.
- Legacy `PUT /api/admin/provider-configs/:kind` remains for compatibility and updates the first row for that kind, or creates one if none exists.
- `apiKey` is write-only. Responses must never include the raw key or encrypted ciphertext.
- `apiKeyLast4` is the only key hint exposed to clients.
- `apiKey` and `clearApiKey` are mutually exclusive.
- `isEnabled=false` disables a database row without deleting it.
- `priority` must be an integer from 1 to 999 for database rows.
- Empty `baseUrl` is allowed only as an explicit provider-mode value. Non-empty `baseUrl` must be a valid URL.
- `extraJson` stores provider-specific fields:
  - `embedding.dimensions`: positive integer.
  - `embedding.version`: non-empty string.
  - `tts.voiceLeft`: non-empty string.
  - `tts.voiceRight`: non-empty string.
- Runtime resolution tries enabled database configs by ascending priority, then tries the matching environment config.
- If a database row has no stored key, runtime must fall back to the matching environment key.
- Embedding fallback candidates must match `EMBEDDING_DIMENSIONS`. Different embedding dimensions require a separate rebuild or compatible group.
- If runtime cannot read `provider_configs`, AI runtime paths must fall back to environment config instead of failing signal creation.
- Admin config read/write paths must expose database failures instead of silently falling back, because operators need to see broken admin storage.
- Saving a new API key requires `PROVIDER_CONFIG_SECRET`; saving non-key fields can work without it.
- Successful updates must write an admin action with `provider_config.update`.
- Successful create/delete must write `provider_config.create` or `provider_config.delete`.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Non-admin request | 403 | FORBIDDEN |
| Unknown `kind` | 400 | VALIDATION_ERROR |
| Invalid non-empty `baseUrl` | 400 | VALIDATION_ERROR |
| Empty `provider` or `model` | 400 | VALIDATION_ERROR |
| Invalid `id` uuid | 400 | VALIDATION_ERROR |
| Row `id` belongs to another `kind` | 404 | NOT_FOUND |
| `apiKey` and `clearApiKey` together | 400 | VALIDATION_ERROR |
| `priority` outside 1-999 | 400 | VALIDATION_ERROR |
| New `apiKey` without `PROVIDER_CONFIG_SECRET` | 400 | VALIDATION_ERROR |
| Runtime DB read failure | n/a | environment fallback |
| Admin DB read/write failure | 500 | database error surfaces |

### 5. Good/Base/Bad Cases

- Good: admin updates embedding `dimensions` and `version`, then the next embedding call reads the new config without process restart.
- Good: admin saves a new key, backend stores AES-256-GCM ciphertext plus last four characters only.
- Good: clearing a database key makes runtime use the environment key when one exists.
- Good: one `chat` config fails at runtime, then the adapter tries the next enabled `chat` config by priority.
- Good: embedding fallback rows with 2048 dimensions are ignored when the runtime vector dimension is 1024.
- Good: reranker provider can be `disabled` and should produce `status=disabled` without breaking matching.
- Base: no database row means the only row for that kind has `source=environment`.
- Bad: return raw API keys, ciphertext, or decrypted key material through admin JSON.
- Bad: cache provider config in process memory in a way that requires a restart after admin edits.
- Bad: swallow admin read/write database errors and pretend env config was saved.
- Bad: use `kind` alone for row update/delete in the multi-provider API.

### 6. Tests Required

- Route test: non-admin requests are rejected.
- Route test: `GET /api/admin/provider-configs` returns database rows plus environment fallback rows and no key material.
- Route test: `PUT /api/admin/provider-configs/:kind` validates kind, URL, provider, model, and key-clear conflict.
- Route test: `POST`, `PUT :id`, and `DELETE :id` create, update, and delete backup rows without returning ciphertext.
- Service test: saving a key stores ciphertext and `apiKeyLast4`, never plaintext.
- Service test: missing `PROVIDER_CONFIG_SECRET` rejects new key storage but allows non-key updates.
- Runtime test: database configs are tried by priority before environment fallback.
- Runtime test: missing database key falls back to environment key.
- Runtime test: runtime database read failure falls back to environment config.
- Runtime test: embedding configs with mismatched dimensions are excluded from fallback candidates.

### 7. Wrong vs Correct

#### Wrong

```typescript
return { providerConfig: { ...row, apiKey: decryptApiKey(row.apiKeyCiphertext) } };
```

#### Correct

```typescript
return {
  providerConfig: {
    kind: row.kind,
    provider: row.provider,
    model: row.model,
    apiKeyLast4: row.apiKeyLast4,
    keyState,
  },
};
```

## Scenario: Shared Web and Android Kotlin API

### 1. Scope / Trigger

- Trigger: Web-first product with Android Kotlin client.
- Any feature used by the UI must be available through Fastify JSON API.
- Server actions may call API helpers internally, but must not become the only implementation path.

### 2. Signatures

Core API signatures:

```text
POST   /api/auth/login
POST   /api/auth/login/verify
POST   /api/auth/frequency/save
POST   /api/auth/frequency/login
POST   /api/auth/frequency/suggestions
POST   /api/auth/logout
GET    /api/me
GET    /api/channels
GET    /api/channels/:channel/voices
POST   /api/voices
GET    /api/voices/:id
DELETE /api/voices/:id
POST   /api/voices/:id/resonance
DELETE /api/voices/:id/resonance
GET    /api/voices/:id/echoes
POST   /api/voices/:id/echoes
GET    /api/bottles/summary
POST   /api/bottles
POST   /api/bottles/open
POST   /api/bottles/:id/echoes
POST   /api/bottles/:id/continue
POST   /api/bottles/:id/keep
GET    /api/radio-report/latest
POST   /api/frontend-diagnostics/errors
POST   /api/reports
POST   /api/images
GET    /api/images/:id
POST   /api/translate
POST   /api/signals
GET    /api/signals/hall
GET    /api/signals/:id
GET    /api/signals/:id/audio
GET    /api/signals/:id/audio/:side
POST   /api/signals/:id/resonance
DELETE /api/signals/:id/resonance
GET    /api/signals/:id/echoes
POST   /api/signals/:id/echoes
POST   /api/ai/classify
POST   /api/ai/tags
POST   /api/ai/moderate
POST   /api/ai/content-analysis
POST   /api/ai/echo-suggestion
GET    /api/admin/overview
GET    /api/admin/voices
PATCH  /api/admin/voices/:id/status
GET    /api/admin/reports
PATCH  /api/admin/reports/:id
GET    /api/admin/users
PATCH  /api/admin/users/:id/status
GET    /api/admin/ai-evaluation
GET    /api/admin/ai-signals
POST   /api/admin/ai-signals/generate
PATCH  /api/admin/ai-signals/:id/keep
DELETE /api/admin/ai-signals/:id
```

### 3. Contracts

Success response:

```json
{
  "data": {},
  "meta": {
    "requestId": "req_123"
  }
}
```

Error response:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request.",
    "details": {}
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

Environment keys:

```text
DATABASE_URL
REDIS_URL
SESSION_SECRET
WEB_ORIGIN
AUTH_LOGIN_CHALLENGE_TTL_SECONDS
AUTH_LOGIN_CODE_RESPONSE
AUTH_LOGIN_DELIVERY_MODE
AUTH_LOGIN_MAX_VERIFY_ATTEMPTS
AUTH_LOGIN_REQUEST_COOLDOWN_SECONDS
AI_PROVIDER
AI_BASE_URL
AI_MODEL
AI_API_KEY
AI_FAST_PROVIDER
AI_FAST_BASE_URL
AI_FAST_MODEL
AI_FAST_API_KEY
IMAGE_MAX_BYTES
IMAGE_STORAGE_DIR
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Invalid request body | 400 | VALIDATION_ERROR |
| Missing session | 401 | UNAUTHORIZED |
| Non-admin uses admin API | 403 | FORBIDDEN |
| Resource not found | 404 | NOT_FOUND |
| Duplicate resonance | 409 | CONFLICT |
| Daily limit reached | 429 | RATE_LIMITED |
| AI provider unavailable | 503 | AI_UNAVAILABLE |
| Unexpected failure | 500 | INTERNAL_ERROR |

### 5. Good/Base/Bad Cases

- Good: Web and Android Kotlin client both call `POST /api/voices` to create a voice.
- Base: Next.js server component calls `GET /api/channels/:channel/voices` for initial render.
- Bad: Next.js writes directly to the database for voice creation while Android Kotlin client uses API.

### 6. Tests Required

- Route tests for status code and JSON shape.
- Auth tests for protected and admin routes.
- Contract tests for request validation and enum values.
- Regression tests for duplicate resonance and daily limits.


## Scenario: Expression Translation API Contracts

Legacy note: this scenario belongs to the old unsent-expression direction. Do not use it for new Unfreq core work unless a task explicitly says to maintain a legacy endpoint. New signal creation uses `POST /api/signals` from `Scenario: Unfreq Resonance Signal API Contracts`.

### 1. Scope / Trigger

- Trigger: Web and future Android clients need the same unsent-expression translation endpoint.
- Applies to intent analysis, recommended expression, risk highlights, reaction simulation, risk scores, fallback status, and publish eligibility.
- The API may use an AI provider internally, but clients must only consume validated JSON.

### 2. Signatures

API:

```text
POST /api/translate
```

### 3. Contracts

Request:

```json
{
  "text": "我想告诉室友别老临时改计划但又怕显得小气",
  "target": "室友",
  "goal": "表达边界"
}
```

- `text`: trimmed string, 5-1000 chars.
- `target`: optional enum `朋友 | 室友 | 老师 | 家人 | 恋人 | 同学 | 同事 | 其他`.
- `goal`: optional enum `解释 | 道歉 | 拒绝 | 求助 | 表达边界 | 澄清误会 | 结束关系 | 其他`.

Response:

```json
{
  "data": {
    "translation": {
      "id": "uuid",
      "originalText": "original text",
      "fallback": false,
      "intent": {
        "target": "室友",
        "goal": "表达边界",
        "emotion": "担心被认为计较",
        "hiddenNeed": "希望自己的时间安排被尊重",
        "risk": "LOW|MEDIUM|HIGH"
      },
      "recommended": "recommended expression",
      "riskHighlights": [
        {
          "text": "短语",
          "reason": "why risky",
          "severity": "LOW|MEDIUM|HIGH",
          "suggestion": "how to change"
        }
      ],
      "reasoning": "why it works",
      "alternatives": {
        "gentle": "gentle version",
        "direct": "direct version",
        "formal": "formal version"
      },
      "reaction": {
        "positive": "likely positive response",
        "neutral": "likely neutral response",
        "defensive": "likely defensive response"
      },
      "riskScores": {
        "before": {
          "aggression": 0,
          "misunderstanding": 0,
          "emotionalIntensity": 0,
          "clarity": 0,
          "repairPotential": 0
        },
        "after": {
          "aggression": 0,
          "misunderstanding": 0,
          "emotionalIntensity": 0,
          "clarity": 0,
          "repairPotential": 0
        }
      },
      "publish": {
        "allowed": true,
        "reason": "why"
      }
    }
  }
}
```

Risk score values are integers from 0 to 100. `publish.allowed` must be false when `intent.risk` is `HIGH`.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Invalid text length or enum | 400 | VALIDATION_ERROR |
| Provider returns non-JSON | 200 | fallback result |
| Provider JSON misses fields | 200 | fallback result |
| Provider exceeds 5s total timeout | 200 | fallback result |
| Provider unavailable after retries | 200 | fallback result |
| Unexpected failure outside AI path | 500 | INTERNAL_ERROR |

### 5. Good/Base/Bad Cases

- Good: route validates request, service retries provider failures twice, zod-validates output, and returns fallback in the same DTO shape.
- Good: high-risk translations return `publish.allowed: false`; publishing decisions are not left to frontend copy.
- Base: `riskHighlights.text` may not exist in `originalText`; clients must render safely.
- Bad: frontend or Android calls the AI provider directly.
- Bad: route returns raw provider text or a partial object.

### 6. Tests Required

- Route test: success response uses shared envelope and full translation object.
- Route test: invalid `text`, `target`, or `goal` returns `400 VALIDATION_ERROR`.
- Service test: provider invalid JSON/schema retries twice, then returns fallback.
- Service test: 5s timeout returns fallback.
- Service test: `HIGH` risk sets `publish.allowed` false.

### 7. Wrong vs Correct

#### Wrong

```typescript
return reply.send(await fetch(aiProviderUrl, { body: request.body as string }));
```

#### Correct

```typescript
const translation = await translateExpression(parsedBody.data);
const response = translateResponseSchema.parse({ translation });
return sendData(reply, request, response);
```

## Scenario: Signal Domain API Contracts

Legacy note: this older signal program contract is superseded by
`Scenario: Pending Signal Emission API` and `Scenario: Unfreq Resonance Signal
API Contracts` for new work. The `generate`/draft-publish split below is
historical; current owner publish confirmation is defined in the pending signal
scenario above. Do not add new fingerprint/program, echo thread, legacy
publish-draft, or double-channel audio behavior to the core product.

### 1. Scope / Trigger

- Trigger: same-frequency signal radio needs stable storage and public JSON APIs.
- Applies to draft generation, explicit publish, hall list, detail read, resonance, and echoes.
- AI quality can improve later; MVP still returns valid structured fallback data.

### 2. Signatures

API:

```text
POST /api/signals/generate
POST /api/signals/:id/publish
GET  /api/signals/hall
GET  /api/signals/:id
POST /api/signals/:id/resonance
DELETE /api/signals/:id/resonance
GET  /api/signals/:id/echoes
POST /api/signals/:id/echoes
```

Database:

```text
signals(
  id,
  author_id nullable,
  guest_key nullable,
  raw_input,
  fingerprint_json,
  program_json,
  match_signal_id nullable,
  status,
  audio_status,
  audio_error nullable,
  audio_storage_key_left nullable,
  audio_storage_key_right nullable,
  audio_url_left nullable,
  audio_url_right nullable,
  resonance_count,
  echo_count,
  created_at,
  updated_at
)

signal_echoes(
  id,
  signal_id,
  author_id nullable,
  guest_key nullable,
  content,
  status,
  created_at,
  updated_at
)

signal_resonances(
  id,
  signal_id,
  actor_key,
  author_id nullable,
  guest_key nullable,
  created_at
)
```

### 3. Contracts

`POST /api/signals/generate` request:

```json
{
  "rawInput": "今晚想把这段信号留在这里。",
  "guestKey": "browser-generated-key"
}
```

Response:

```json
{
  "data": {
    "signal": {
      "id": "uuid",
      "rawInput": "今晚想把这段信号留在这里。",
      "status": "DRAFT",
      "fingerprint": {
        "title": "今晚想把这段信号留在这里",
        "theme": "signal",
        "primaryEmotion": "未命名情绪",
        "secondaryEmotions": ["等待", "迟疑"],
        "narrativeConflict": "想靠近别人，又担心打扰别人",
        "hiddenNeed": "希望被理解",
        "energy": 0.4,
        "openness": 0.4,
        "frequency": "SIG-ABC123",
        "signalType": "life-fragment",
        "tags": ["signal"],
        "riskLevel": "LOW"
      },
      "program": {
        "programTitle": "同频试播",
        "hostIntro": "这里是 SIG-ABC123。",
        "leftSignalSummary": "今晚想把这段信号留在这里。",
        "rightSignalSummary": "另一段同频信号摘要。",
        "bridgeNarration": "等待下一段同频回应。",
        "closingQuestion": "如果有人听见，会怎么回应？",
        "shareQuote": "我留下了一段信号。"
      },
      "audioStatus": "PENDING",
      "matchSignalId": "uuid",
      "matches": [
        {
          "id": "uuid-or-seed-id",
          "title": "夜里的空白座位",
          "summary": "有人把一整天的沉默留在夜里。",
          "primaryEmotion": "孤独",
          "signalType": "life-fragment",
          "tags": ["孤独", "夜晚"],
          "score": 125,
          "source": "PUBLISHED"
        }
      ]
    }
  }
}
```

Generation returns up to 3 same-frequency matches. `source` is `PUBLISHED` for persisted public signals or `SEEDED` for demo fallback matches. `matchSignalId` is present only when the primary match is a persisted `PUBLISHED` signal; seeded matches must not be stored as `match_signal_id`.

Signal audio status:

```text
PENDING -> service-side TTS may still generate mp3 files
READY   -> both left and right audio URLs are present
FAILED  -> text program remains usable; client may use browser speech fallback
```

`GET /api/signals/:id/audio` response:

```json
{
  "data": {
    "audio": {
      "signalId": "uuid",
      "audioStatus": "READY",
      "audioUrlLeft": "/api/signals/uuid/audio/left",
      "audioUrlRight": "/api/signals/uuid/audio/right"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/signals/:id/audio/:side` accepts only `left` or `right` and streams
`audio/mpeg` when the signal audio is `READY`. The audio files are stored by the
API service and served through Fastify; the API container must not write into
the Web container's `public/` directory.

Public hall and detail endpoints return only `PUBLISHED` signals.
Public echo list returns only `PUBLISHED` echoes for a `PUBLISHED` signal, ordered
by `createdAt` ascending.

`POST /api/signals/:id/resonance` and `DELETE /api/signals/:id/resonance` accept:

```json
{
  "guestKey": "browser-generated-key"
}
```

When a session frequency is present, the server uses the session identity. When no
session exists, `guestKey` is required. The pair `(signal_id, actor_key)` is
unique, so repeated resonance does not increase the count twice.

Response:

```json
{
  "data": {
    "resonance": {
      "hasResonated": true,
      "resonanceCount": 1
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/signals/:id/echoes` response:

```json
{
  "data": {
    "echoes": [
      {
        "id": "uuid",
        "signalId": "uuid",
        "content": "我也在这个频率附近。",
        "status": "PUBLISHED",
        "authorFreqCode": "FM-27.51",
        "guestKey": "browser-generated-key",
        "createdAt": "2026-05-13T00:00:00.000Z"
      }
    ]
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Invalid raw input or echo body | 400 | VALIDATION_ERROR |
| AI provider unavailable during generation | 201 | deterministic fallback output |
| Generated fingerprint has `riskLevel=HIGH` | 201 | signal status `REVIEWING` |
| Published/hidden/deleted draft publish attempt | 409 | CONFLICT |
| Missing signal for detail/publish/echo | 404 | NOT_FOUND |
| Audio status requested with no configured TTS key | 200 | audio.status `FAILED`, text fallback allowed |
| Audio download before `READY` | 409 | CONFLICT |
| Audio download side outside `left/right` | 400 | VALIDATION_ERROR |
| Audio storage key missing for `READY` signal | 404 | NOT_FOUND |
| Missing guest key for anonymous resonance | 400 | VALIDATION_ERROR |
| Missing signal for detail/publish/echo/resonance | 404 | NOT_FOUND |
| Muted or banned session publishes/echoes/resonates | 403 | FORBIDDEN |

### 5. Good/Base/Bad Cases

- Good: generate creates a `DRAFT`; publish is a separate explicit call.
- Good: high-risk generation stores `REVIEWING`, so publish returns conflict until moderation changes status.
- Good: Top 3 matching ranks same `signalType`, overlapping tags, then same primary emotion.
- Good: hall/detail filter `PUBLISHED` in repository queries.
- Good: echo list verifies the parent signal is `PUBLISHED` and filters echoes to
  `PUBLISHED` in SQL before returning the shared envelope.
- Good: server-side signal audio generation is best-effort, deduplicated per
  signal in the API process, and never blocks signal text generation.
- Good: audio URLs point at API routes such as `/api/signals/:id/audio/left`,
  not at Web public files.
- Good: resonance insert/delete is idempotent and updates `signals.resonance_count`
  inside a transaction.
- Good: Zod parses response DTOs before route sends them.
- Base: fallback generation is deterministic enough for local demo without AI keys.
- Base: seeded matches can fill local/demo output when no published candidate exists.
- Base: missing `TTS_API_KEY` keeps signal text usable and lets Web use browser
  speech fallback.
- Bad: expose `DRAFT`, `HIDDEN`, or `DELETED` signals through public hall/detail.
- Bad: expose echoes for a hidden/deleted signal, or filter signal/echo visibility
  in the frontend after fetching private rows.
- Bad: write seeded match ids into `signals.match_signal_id`.
- Bad: store provider raw output in `fingerprint_json` or `program_json`.
- Bad: require TTS success before returning `POST /api/signals/generate`.
- Bad: write mp3 files from the API container into `apps/web/public`.
- Bad: increment `signals.resonance_count` without a unique actor record.

### 6. Tests Required

- Route test: generate success creates a draft with fingerprint and program.
- Route test: invalid raw input returns `400 VALIDATION_ERROR`.
- Service test: matching returns stable Top 3 by type, tag overlap, and emotion.
- Service test: seeded fallback returns demo matches without `match_signal_id`.
- Service test: no-match input generates a solo program.
- Service test: high-risk fingerprint creates `REVIEWING`.
- AI service test: invalid provider output records fallback for fingerprint/program.
- Route test: publish moves draft to `PUBLISHED`.
- Route test: hall and detail only expose published signals.
- Route test: audio status returns envelope and pending/failed/ready states.
- Route test: audio download returns `409 CONFLICT` before ready and streams
  `audio/mpeg` after ready.
- Unit test: TTS adapter sends DashScope `run-task`, `continue-task`, and
  `finish-task` events and rejects unconfigured providers before opening a
  socket.
- Route test: signal echo list returns published echoes for a published signal.
- Route test: echo creation increments signal echo flow and returns envelope.
- Route test: repeated signal resonance and repeated delete keep count idempotent.
- Route test: anonymous resonance without `guestKey` returns `400 VALIDATION_ERROR`.
- Route test: hidden signal detail returns `404 NOT_FOUND`.

### 7. Wrong vs Correct

#### Wrong

```typescript
const rows = await db.select().from(signals);
return rows.filter((signal) => signal.status !== "HIDDEN");
```

#### Correct

```typescript
return db
  .select(signalSelection)
  .from(signals)
  .where(and(
    eq(signals.status, "PUBLISHED"),
    eq(signals.generationStatus, "COMPLETED"),
  ))
  .orderBy(desc(signals.createdAt));
```

#### Wrong

```typescript
const echoes = await db.select().from(signalEchoes);
return echoes.filter((echo) => echo.signalId === id);
```

#### Correct

```typescript
return db
  .select(signalEchoSelection)
  .from(signalEchoes)
  .innerJoin(signals, eq(signalEchoes.signalId, signals.id))
  .where(
    and(
      eq(signalEchoes.signalId, id),
      eq(signalEchoes.status, "PUBLISHED"),
      eq(signals.status, "PUBLISHED"),
      eq(signals.generationStatus, "COMPLETED"),
    ),
  )
  .orderBy(asc(signalEchoes.createdAt));
```

## Scenario: AI Signal Content Ecosystem

### 1. Scope / Trigger

- Trigger: cold-start signal content, AI-generated signal batches, seed library initialization, and admin review of AI signals.
- Applies to `signals.source`, quality self-check, persona-backed AI interactions, seed commands, and admin APIs.
- This is demo infrastructure. In-process scheduling is allowed only behind a replaceable module and must not claim cross-restart delivery.

### 2. Signatures

API:

```text
GET    /api/admin/ai-signals?source=ai_generated|seed|human
POST   /api/admin/ai-signals/generate
PATCH  /api/admin/ai-signals/:id/keep
DELETE /api/admin/ai-signals/:id
```

Command:

```text
npm --workspace @unfreq/api run signals:seed
```

Database:

```text
signals.source signal_source not null default human
signals.quality_score integer nullable
signals.quality_reason text nullable
signals.is_kept boolean not null default false
target_type includes SIGNAL
```

Persona config:

```text
apps/api/src/config/personas.json
personas[].id
personas[].fmName
personas[].age
personas[].occupation
personas[].currentConcern
personas[].style
personas[].commonScenes
```

### 3. Contracts

Admin AI signal DTOs reuse `SignalDetail` and must include:

```text
source: human | ai_generated | seed
qualityScore?: 1..5
qualityReason?: string
isKept: boolean
```

`POST /api/admin/ai-signals/generate` accepts:

```json
{
  "count": 5
}
```

`count` is optional and clamped to `1..10`. Omitted count generates `5..10` signals. The generator cycles through configured personas when the requested count is greater than the persona count.

Generation rules:

- `generateSignalContentWithFallback` creates 30-100 Chinese characters from a fixed persona.
- `scoreSignalQualityWithFallback` runs before insert.
- Score below `3` is discarded, with at most 3 attempts per requested signal.
- Accepted AI signals are stored as `PUBLISHED` with `source='ai_generated'`.
- Seed initialization uses `source='seed'` and a stricter score threshold of `4`.
- Generated and seed signals run the same fingerprint, program, match, audio, and embedding flow as human signals.
- Seed initialization writes embeddings synchronously when available so semantic matching can use the library immediately.

Interaction rules:

- Publishing or generating a `PUBLISHED` signal schedules AI interactions.
- Production starts after 5 minutes and spreads 2-5 resonances plus 1-2 echoes across a 24-hour window.
- Development may use short delays for local feedback.
- Persona identity uses stable keys: `ai-persona:<persona.id>`.
- Echoes are 30-50 characters and only express resonance or empathy. They must not give advice.
- In-process timers must be cleared on Fastify `onClose` and skipped in `NODE_ENV=test`.

### 4. Validation & Error Matrix

| Condition | Status / Behavior |
|-----------|-------------------|
| Missing session on admin API | 401 UNAUTHORIZED |
| Non-admin frequency | 403 FORBIDDEN |
| Bad source query or keep body | 400 VALIDATION_ERROR |
| Missing signal for keep/delete | 404 NOT_FOUND |
| Low quality AI generation | discard and retry, no insert |
| All generation attempts rejected | return accepted count below requested count |
| Existing seed raw input | skip, do not duplicate |
| Seed quality below 4 | reject and exit script non-zero |
| Embedding provider unavailable | keep created signal; fallback matching still works |
| Process restarts during timer window | interactions may be lost until a durable worker replaces in-process scheduling |

### 5. Good/Base/Bad Cases

- Good: admin APIs require the same admin session guard and write `admin_actions` for keep/delete.
- Good: AI-generated content is inserted only after quality scoring.
- Good: seed data lives in a script/module, not in migrations.
- Good: in-process timers live in a small scheduler module and can be replaced by BullMQ later.
- Base: local demos with no AI or embedding key still use deterministic fallbacks and keyword/tag matching.
- Bad: create seed rows inside schema migrations.
- Bad: mark AI-generated rows as human content.
- Bad: schedule timers in tests or without shutdown cleanup.
- Bad: claim at-least-once delivery for in-memory interaction scheduling.

### 6. Tests Required

- Service test: AI generation inserts `source='ai_generated'`, quality score, and `PUBLISHED` status after quality check.
- Service test: low quality content retries up to 3 times and does not insert rejected content.
- Service test: request count above persona count still creates the requested number by cycling personas.
- Service test: seed command path skips existing raw inputs and uses score threshold 4.
- Route test: admin list/generate/keep/delete require admin auth and return shared envelopes.
- Route test: keep/delete writes persisted admin actions.
- Type-check: backend and Web DTOs agree on `source`, `qualityScore`, `qualityReason`, and `isKept`.

### 7. Wrong vs Correct

#### Wrong

```typescript
await createSignalDraft({
  rawInput,
  source: "human",
  status: "PUBLISHED",
});
```

#### Correct

```typescript
const quality = await scoreSignalQualityWithFallback({ content: rawInput });
if (quality.result.score >= 3) {
  await createSignalDraft({
    rawInput,
    qualityReason: quality.result.reason,
    qualityScore: quality.result.score,
    source: "ai_generated",
    status: "PUBLISHED",
  });
}
```

#### Wrong

```typescript
setTimeout(() => generateSignalInteractions(signal.id), delayMs);
```

#### Correct

```typescript
if (!env.isTest) {
  const stopScheduler = startAiSignalScheduler(app.log);
  app.addHook("onClose", async () => stopScheduler());
}
```

## Scenario: Unfreq Admin Console Contracts

### 1. Scope / Trigger

- Trigger: `/admin` operations dashboard, signal governance, frequency status changes, AI observability, galaxy operation status, and seed library tooling.
- Applies to Web admin UI and future internal clients through the same Fastify JSON API.
- Retired `voices` admin routes are not part of the default API. The primary console must use `signals`, `frequencies`, `reports`, `ai_reviews`, and `admin_actions`.

### 2. Signatures

API:

```text
GET   /api/admin/overview
GET   /api/admin/signals?source=ai_generated|seed|human&status=PUBLISHED|REVIEWING|HIDDEN|DELETED|DRAFT
PATCH /api/admin/signals/:id/status
GET   /api/admin/frequencies?status=ACTIVE|MUTED|BANNED
PATCH /api/admin/frequencies/:id/status
GET   /api/admin/reports?status=PENDING|RESOLVED|REJECTED
PATCH /api/admin/reports/:id
GET   /api/admin/ai-reviews
GET   /api/admin/ai-evaluation
GET   /api/admin/ai-signals?source=ai_generated|seed|human
POST  /api/admin/ai-signals/generate
PATCH /api/admin/ai-signals/:id/keep
DELETE /api/admin/ai-signals/:id
```

Database:

```text
signals
frequencies
reports
ai_reviews
admin_actions
```

### 3. Contracts

`GET /api/admin/overview` returns counts for:

```text
signals.total/published/reviewing/hidden/deleted/draft
reports.pending/resolved/rejected
frequencies.active/muted/banned
aiOperations.chatFallbacks/embeddingFailures/rerankerFailures/ttsFailures
capabilities.chat/embedding/reranker/tts = configured | degraded | disabled
galaxy.missingCoordinates
```

`GET /api/admin/signals` returns compact signal rows:

```text
id
frequencyCode?
seedText
expandedText
source
status
resonanceCount
audioStatus
embedding.status = ready | missing
embedding.provider/model/dimensions/version?
galaxyPositionStatus = ready | missing
qualityScore?
qualityReason?
createdAt
updatedAt
```

Admin signal status changes must update `signals.status` and write `admin_actions` with:

```text
targetType=SIGNAL
targetId=<signal id>
action=signal.status.update
details.status=<new status>
```

Admin frequency status changes must update both the canonical `frequencies` row and the legacy `users` mirror while auth still reads `users`.

### 4. Validation & Error Matrix

| Condition | Status | Code / Behavior |
|-----------|--------|-----------------|
| Missing session on admin API | 401 | UNAUTHORIZED |
| Non-admin frequency | 403 | FORBIDDEN |
| Bad query enum or status body | 400 | VALIDATION_ERROR |
| Missing signal/frequency | 404 | NOT_FOUND |
| Admin mutation succeeds | 200 | shared envelope and `admin_actions` row |
| AI capability has no configured key/base URL | 200 | capability `disabled` |
| AI operation has fallback/failure count | 200 | capability may be `degraded` |

### 5. Good/Base/Bad Cases

- Good: signal governance uses `/api/admin/signals`, not the seed-only `/api/admin/ai-signals` endpoint.
- Good: frequency status updates keep `frequencies.status` and `users.status` aligned until auth fully migrates.
- Good: AI observability shows provider/model/status summaries and never returns API keys or raw provider payloads.
- Base: historical voice rows may still exist in the database, but the admin console does not expose a compatibility section.
- Bad: make `/admin` compute metrics from old voices/bottles as the primary dashboard.
- Bad: hide a signal only in the frontend without writing `admin_actions`.

### 6. Tests Required

- Route test: admin overview returns signal/frequency/AI/galaxy metrics in the shared envelope.
- Route test: `GET /api/admin/signals` supports source and status filters.
- Route test: `PATCH /api/admin/signals/:id/status` writes a `SIGNAL` admin action.
- Route test: `PATCH /api/admin/frequencies/:id/status` writes a `USER` admin action and returns the frequency DTO.
- Frontend typecheck: `AdminDashboardData` matches all admin endpoint DTOs.

### 7. Wrong vs Correct

#### Wrong

```typescript
await apiClient.patch(`/api/admin/voices/${id}/status`, { status: "HIDDEN" });
```

#### Correct

```typescript
await apiClient.patch(`/api/admin/signals/${id}/status`, { status: "HIDDEN" });
```

## Scenario: Frontend Diagnostics

### 1. Scope / Trigger

- Trigger: Next.js client runtime errors, unhandled promise rejections, and App Router error boundaries.
- Applies to Web diagnostics only. The endpoint is unauthenticated and must not receive session cookies, email addresses, or raw voice/echo/bottle content.
- Diagnostics are logged for incident review, not stored as product data.

### 2. Signatures

API:

```text
POST /api/frontend-diagnostics/errors
```

### 3. Contracts

Request:

```json
{
  "source": "client-runtime",
  "route": "/voices/uuid",
  "name": "TypeError",
  "message": "Cannot read properties of undefined.",
  "correlationId": "web_abc123",
  "requestId": "req_123",
  "digest": "NEXT_DIGEST"
}
```

Response:

```json
{
  "data": {
    "accepted": true,
    "correlationId": "web_abc123"
  },
  "meta": {
    "requestId": "req_456"
  }
}
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Valid diagnostic | 202 | accepted |
| Extra fields such as cookies | 400 | VALIDATION_ERROR |
| Route includes query string or absolute URL | 400 | VALIDATION_ERROR |
| Message, name, digest, or request id too long | 400 | VALIDATION_ERROR |

### 5. Good/Base/Bad Cases

- Good: browser sends only route path, error name/message, correlation id, optional API request id, and optional Next digest.
- Base: diagnostic send failure is swallowed by the UI after a local development console warning.
- Bad: sending cookies, authorization headers, email addresses, full user content, stack traces, or arbitrary extra fields.

### 6. Tests Required

- Route accepts unauthenticated diagnostics and returns the shared envelope.
- Route rejects extra fields that could carry cookies or content.
- Route rejects route values with query strings.

## Scenario: AI Emotion Radio Contracts

### 1. Scope / Trigger

- Trigger: AI emotion features that affect Web, API, database, admin evidence, and Android Kotlin client.
- Applies to content emotion analysis, frequency emotion profile updates, semantic bottle matching, daily radio report, and admin AI evaluation.
- These features must expose stable JSON APIs; do not implement them as Web-only or screenshot-only logic.

### 2. Signatures

API:

```text
POST /api/ai/content-analysis
GET  /api/radio-report/latest
GET  /api/admin/ai-evaluation
POST /api/bottles/open
```

Database:

```text
users.emotion_profile jsonb
voices.ai_analysis jsonb
bottles.ai_analysis jsonb
daily_radio_reports(
  id,
  report_date,
  status,
  top_emotions,
  channel_highlights,
  summary,
  radio_line,
  source_counts,
  generated_at,
  created_at,
  updated_at
)
```

AI review types:

```text
content-analysis
daily-radio-report
classify
tags
moderate
echo-suggestion
signal-fingerprint
signal-program
```

### 3. Contracts

`POST /api/ai/content-analysis` request:

```json
{
  "content": "今晚有点孤独，也有点想念。"
}
```

`POST /api/ai/content-analysis` response:

```json
{
  "data": {
    "result": {
      "summary": "今晚有点孤独，也有点想念。",
      "emotions": ["孤独", "想念"],
      "intensity": 0.72,
      "riskLevel": "LOW",
      "reason": "未发现明显风险信号。",
      "keywords": ["孤独", "想念"]
    },
    "reviewId": "uuid"
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/radio-report/latest` response:

```json
{
  "data": {
    "report": {
      "id": "uuid",
      "date": "2026-05-13",
      "status": "READY",
      "topEmotions": ["孤独", "想念"],
      "channelHighlights": [
        {
          "channel": "MIDNIGHT",
          "count": 2,
          "summary": "深夜频段里，疲惫和想念更密集。"
        }
      ],
      "summary": "今天更多人在谈孤独、想念。",
      "radioLine": "今晚的频率偏低，但仍有人回应。",
      "sourceCounts": {
        "voices": 2,
        "bottles": 1,
        "total": 3
      },
      "generatedAt": "2026-05-13T10:00:00.000Z"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

Daily report status:

```text
READY -> enough anonymous aggregate signals; page may show trends
EMPTY -> not enough signals; page must show low-content state
```

Privacy contract:

- Daily report input to AI uses aggregate counts and top emotions only.
- Daily report response must not include raw voice/bottle content, author ids, emails, `freqCode`, or single-user excerpts.
- Semantic bottle matching may read `users.emotion_profile` and candidate `bottles.ai_analysis`, but the public response remains the existing bottle detail contract.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Empty content analysis body | 400 | VALIDATION_ERROR |
| Direct AI provider unavailable on `/api/ai/content-analysis` | 503 | AI_UNAVAILABLE |
| Voice/bottle create AI unavailable | create still succeeds through fallback or moderation state |
| Daily report has fewer than 3 same-day signals | 200 | report.status = EMPTY |
| Daily report AI summary unavailable | 200 | rule fallback persisted |
| Non-admin reads `/api/admin/ai-evaluation` | 403 | FORBIDDEN |

### 5. Good/Base/Bad Cases

- Good: `POST /api/voices` stores `ai_analysis` and updates `users.emotion_profile` inside the API write path.
- Good: `POST /api/bottles/open` scores candidates with semantic signal, emotion compatibility, freshness, and safety weight, then falls back to random if signals are missing.
- Good: `GET /api/radio-report/latest` persists the generated report once per date and reuses it on later reads.
- Good: `GET /api/admin/ai-evaluation` uses fixed samples and deterministic local fallback, not private production text.
- Base: Empty `AI_BASE_URL` returns deterministic fallback output.
- Bad: calling an AI provider from Next.js or Android Kotlin client.
- Bad: including raw voice/bottle content in a daily report response.
- Bad: generating the daily report on every page load without checking `daily_radio_reports`.

### 6. Tests Required

- AI route test: validates `POST /api/ai/content-analysis` envelope and invalid-body errors.
- Service test: provider failure records fallback for business callers.
- Voice/bottle tests: creation persists AI analysis and keeps content creation alive when provider fallback is used.
- Matching test: semantic, emotion, freshness, and safety scoring affect chosen bottle.
- Daily report tests: existing report reuse, generation, empty state, and AI fallback persistence.
- Admin evaluation tests: classification accuracy, tag Top-K hit rate, moderation recall, match Top-K relevance, baselines, and ablations.
- Frontend typecheck: `DailyRadioReport` and `AdminAiEvaluation` DTOs stay aligned with API fields.

### 7. Wrong vs Correct

#### Wrong

```typescript
// Leaks raw content into a public aggregate report.
const rows = await db.select({ content: voices.content }).from(voices);
return sendData(reply, request, { report: { excerpts: rows } });
```

#### Correct

```typescript
const signals = await listDailyRadioReportSignals({ dayStart, dayEnd });
const report = await buildReportFromAggregateSignals(signals);
return sendData(reply, request, { report });
```

#### Wrong

```typescript
// Web-only evidence, not reproducible.
const metrics = [{ label: "效果很好", value: "90%" }];
```

#### Correct

```typescript
const evaluation = await apiClient.get<AdminAiEvaluationResponse>(
  "/api/admin/ai-evaluation",
);
```

## Scenario: Image Hosting API

### 1. Scope / Trigger

- Trigger: product features need user-uploaded images that Web and Android Kotlin client can both consume.
- Applies to image upload, metadata persistence, local storage, and public image retrieval.
- Upload uses the shared JSON API. Raw image retrieval is the only binary response exception.

### 2. Signatures

API:

```text
POST /api/images
GET  /api/images/:id
```

Database:

```text
images(id, uploader_id, mime_type, size_bytes, storage_key, alt_text, created_at, updated_at)
daily_limits(id, user_id, limit_type='IMAGE_UPLOAD', limit_date, count, updated_at)
```

Environment:

```text
IMAGE_MAX_BYTES=2097152
IMAGE_STORAGE_DIR=storage/images
```

### 3. Contracts

`POST /api/images` request:

```json
{
  "contentBase64": "<base64 image bytes>",
  "mimeType": "image/png",
  "altText": "optional short text"
}
```

Supported MIME types:

```text
image/gif
image/jpeg
image/png
image/webp
```

`POST /api/images` response:

```json
{
  "data": {
    "image": {
      "id": "uuid",
      "url": "/api/images/uuid",
      "mimeType": "image/png",
      "sizeBytes": 1024,
      "altText": "optional short text",
      "createdAt": "2026-05-12T00:00:00.000Z"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/images/:id` returns raw image bytes with `Content-Type`, `Content-Length`, and immutable cache headers. Error responses still use the shared JSON error envelope.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Missing session | 401 | UNAUTHORIZED |
| Frequency is muted or banned | 403 | FORBIDDEN |
| Invalid body or base64 | 400 | VALIDATION_ERROR |
| Unsupported MIME type | 400 | VALIDATION_ERROR |
| MIME does not match image signature | 400 | VALIDATION_ERROR |
| Image exceeds `IMAGE_MAX_BYTES` | 400 | VALIDATION_ERROR |
| Daily image upload limit reached | 429 | RATE_LIMITED |
| Image id not found | 404 | NOT_FOUND |

### 5. Good/Base/Bad Cases

- Good: Web and Android Kotlin client call `POST /api/images`, then store or render the returned `image.url`.
- Base: `GET /api/images/:id` streams bytes from the API service through Caddy.
- Bad: Frontend writes directly to local storage or assumes a third-party image host URL format.

### 6. Tests Required

- Upload rejects missing session with `UNAUTHORIZED`.
- Upload rejects inactive frequency with `FORBIDDEN`.
- Upload rejects invalid base64 and MIME signature mismatch with `VALIDATION_ERROR`.
- Upload rejects oversized images with `VALIDATION_ERROR`.
- Upload rejects daily limit overflow with `RATE_LIMITED` and removes the staged file.
- Successful upload returns the shared envelope and a URL.
- Returned URL serves bytes with the stored MIME type.
- Missing image returns `NOT_FOUND`.

### 7. Wrong vs Correct

#### Wrong

```typescript
await writeFile(`public/uploads/${name}`, bytes);
return { url: `/uploads/${name}` };
```

#### Correct

```typescript
const image = await createImage(frequency, {
  contentBase64,
  mimeType: "image/png"
});
return sendData(reply, request, { image }, 201);
```

### 7. Wrong vs Correct

#### Wrong

```typescript
// Next.js-only write path.
await db.insert(voices).values(formData);
```

#### Correct

```typescript
await apiClient.post("/api/voices", {
  channel: "MIDNIGHT",
  content,
  moodTags,
  allowEcho: true
});
```

## Scenario: Auth and Frequency Identity

### 1. Scope / Trigger

- Trigger: login, current frequency identity, and protected-route checks.
- Applies to both Next.js Web and Android Kotlin client.
- Web uses the HTTP-only session cookie.
- Android Kotlin client stores the returned `sessionToken` in app-local session storage and sends `Authorization: Bearer <sessionToken>`.

### 2. Signatures

API:

```text
POST /api/auth/login
POST /api/auth/login/verify
POST /api/auth/logout
GET  /api/me
```

Database:

```text
users(id, email, freq_code, role, status, created_at, updated_at)
```

Session:

```text
Cookie: unfreq_session=<token>
Authorization: Bearer <token>
```

### 3. Contracts

`POST /api/auth/login` request:

```json
{
  "email": "name@example.com"
}
```

`POST /api/auth/login` response:

```json
{
  "data": {
    "challenge": {
      "expiresAt": "2026-05-10T00:10:00.000Z",
      "retryAfterSeconds": 60
    },
    "delivery": {
      "mode": "local-log",
      "code": "123456"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`delivery.code` is allowed only when `AUTH_LOGIN_CODE_RESPONSE=enabled` for local/demo use. Production defaults to hiding the code from the response.

`POST /api/auth/login/verify` request:

```json
{
  "email": "name@example.com",
  "code": "123456"
}
```

`POST /api/auth/login/verify` response:

```json
{
  "data": {
    "frequency": {
      "id": "uuid",
      "freqCode": "FM-27.51",
      "role": "USER",
      "status": "ACTIVE",
      "createdAt": "2026-05-10T00:00:00.000Z"
    },
    "isNewFrequency": true,
    "sessionToken": "v1.payload.signature"
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/me` response:

```json
{
  "data": {
    "frequency": {
      "id": "uuid",
      "freqCode": "FM-27.51",
      "role": "USER",
      "status": "ACTIVE",
      "createdAt": "2026-05-10T00:00:00.000Z"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

Session cookie contract:

- Cookie name: `unfreq_session`.
- Cookie flags: `HttpOnly`, `SameSite=Lax`, `Path=/`, `Max-Age=<ttl>`.
- Production adds `Secure`.
- Token version format: `v1.<base64url-payload>.<hmac-signature>`.
- Token lookup must re-read the user row so role/status changes take effect.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Invalid email body | 400 | VALIDATION_ERROR |
| Invalid verification code body | 400 | VALIDATION_ERROR |
| Missing or expired login challenge | 401 | UNAUTHORIZED |
| Wrong verification code | 401 | UNAUTHORIZED |
| Missing or invalid session | 401 | UNAUTHORIZED |
| Banned frequency token | 401 | UNAUTHORIZED |
| FM code space exhausted | 409 | CONFLICT |
| Repeated challenge request during cooldown | 429 | RATE_LIMITED |
| Too many verification attempts | 429 | RATE_LIMITED |
| Unexpected persistence failure | 500 | INTERNAL_ERROR |

### 5. Good/Base/Bad Cases

- Good: `POST /api/auth/login` normalizes email, writes a short-lived Redis challenge, and does not create a user.
- Good: `POST /api/auth/login/verify` consumes the Redis challenge, reuses the existing frequency identity, and returns the same `freqCode`.
- Base: Web relies on the HTTP-only cookie; Android Kotlin client sends `Authorization: Bearer <sessionToken>`.
- Bad: A Next.js page creates a user or frequency code directly in the database.
- Bad: Returning internal challenge storage keys or email hashes to clients.

### 6. Tests Required

- Requesting a login challenge does not create a frequency identity.
- Verification creates a new frequency identity and returns the shared envelope.
- Re-login verification with the same email keeps the original `freqCode`.
- Invalid and expired challenges return controlled errors.
- Repeated challenge requests are rate-limited.
- `/api/me` rejects missing sessions with `UNAUTHORIZED`.
- `/api/me` returns the current `frequency` DTO from a valid cookie or bearer token.
- Invalid email does not create a `users` row.

### 7. Wrong vs Correct

#### Wrong

```typescript
await db.insert(users).values({ email, freqCode: "FM-27.51" });
```

#### Correct

```typescript
const challenge = await authService.requestLoginChallenge(redis, email);
return sendData(reply, request, {
  challenge: challenge.challenge,
  delivery: challenge.delivery
});
```

## Scenario: Channel and Voice System

### 1. Scope / Trigger

- Trigger: channel overview, public voice lists, voice creation, and voice detail.
- Applies to Web now and Android Kotlin client later through the same Fastify JSON API.
- Star map is an aggregate channel, not a direct broadcast target.

### 2. Signatures

API:

```text
GET  /api/channels
GET  /api/channels/:channel/voices
POST /api/voices
GET  /api/voices/:id
```

Database:

```text
voices(id, author_id, channel, content, mood_tags, allow_echo, status,
       resonance_count, echo_count, created_at, updated_at)
daily_limits(id, user_id, limit_type, limit_date, count, updated_at)
```

### 3. Contracts

Channel slugs:

```text
midnight -> MIDNIGHT
unsent   -> UNSENT
starmap  -> STARMAP
```

`GET /api/channels` response:

```json
{
  "data": {
    "channels": [
      {
        "channel": "MIDNIGHT",
        "slug": "midnight",
        "title": "深夜电台",
        "description": "收听凌晨之后无人认领的声音。",
        "todayVoiceCount": 0,
        "totalVoiceCount": 0
      }
    ]
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`POST /api/voices` request:

```json
{
  "channel": "MIDNIGHT",
  "content": "今晚想把这段声音留在这里。",
  "moodTags": ["孤独", "压力"],
  "allowEcho": true
}
```

`POST /api/voices` response:

```json
{
  "data": {
    "voice": {
      "id": "uuid",
      "channel": "MIDNIGHT",
      "freqCode": "FM-27.51",
      "content": "今晚想把这段声音留在这里。",
      "excerpt": "今晚想把这段声音留在这里。",
      "moodTags": ["孤独", "压力"],
      "allowEcho": true,
      "status": "PUBLISHED",
      "resonanceCount": 0,
      "echoCount": 0,
      "createdAt": "2026-05-10T00:00:00.000Z"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Missing session on create | 401 | UNAUTHORIZED |
| Muted or banned frequency creates voice | 403 | FORBIDDEN |
| Invalid channel/content/tags | 400 | VALIDATION_ERROR |
| `STARMAP` used as create target | 400 | VALIDATION_ERROR |
| Daily voice limit reached | 429 | RATE_LIMITED |
| Hidden/deleted voice detail | 404 | NOT_FOUND |

### 5. Good/Base/Bad Cases

- Good: Lists query `PUBLISHED` voices in SQL and join the author `freqCode`.
- Base: Resonance and echo counts start at `0` until the interaction task adds mutations.
- Bad: Frontend filters hidden voices after fetching all statuses.

### 6. Tests Required

- Channel overview returns three channels in product order.
- Channel overview and star map expose no ranking or recommendation fields.
- Voice creation rejects missing session.
- Voice creation rejects `STARMAP` as a direct target.
- Created voice is visible through detail and channel list endpoints.
- Invalid voice id or hidden/deleted voice returns shared error envelope.

### 7. Wrong vs Correct

#### Wrong

```typescript
const allVoices = await db.select().from(voices);
return allVoices.filter((voice) => voice.status === "PUBLISHED");
```

#### Correct

```typescript
return listPublishedVoicesByChannel("MIDNIGHT", 30);
```

## Scenario: Resonance and Echo

### 1. Scope / Trigger

- Trigger: resonance toggle, echo creation/listing, and my-frequency interaction summary.
- Applies to public voice detail, authenticated mutations, and `/api/me` aggregation.
- Echoes are flat and chronological. No nested replies or echo reactions.

### 2. Signatures

API:

```text
POST   /api/voices/:id/resonance
DELETE /api/voices/:id/resonance
GET    /api/voices/:id/echoes
POST   /api/voices/:id/echoes
GET    /api/me
```

Database:

```text
resonances(id, user_id, voice_id, created_at)
echoes(id, voice_id, author_id, content, status, created_at, updated_at)
```

### 3. Contracts

Resonance response:

```json
{
  "data": {
    "resonance": {
      "hasResonated": true,
      "resonanceCount": 1
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

Echo response:

```json
{
  "data": {
    "echo": {
      "id": "uuid",
      "voiceId": "uuid",
      "freqCode": "FM-27.51",
      "content": "这是一段回声。",
      "createdAt": "2026-05-10T00:00:00.000Z"
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

`GET /api/me` additionally returns:

```json
{
  "interactionSummary": {
    "voiceCount": 1,
    "totalResonanceReceived": 3,
    "totalEchoesReceived": 2,
    "recentEchoes": []
  }
}
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Missing session for mutation | 401 | UNAUTHORIZED |
| Duplicate resonance create | 200 | idempotent success |
| Delete absent resonance | 200 | idempotent success |
| Hidden/deleted voice | 404 | NOT_FOUND |
| Echo body empty or too long | 400 | VALIDATION_ERROR |
| Voice has `allowEcho=false` | 409 | CONFLICT |

### 5. Good/Base/Bad Cases

- Good: `resonances(user_id, voice_id)` has a unique index and count updates happen only when insert/delete changes a row.
- Base: Echo lists are ordered by `created_at` ascending.
- Bad: Incrementing resonance count before checking whether the resonance row already exists.

### 6. Tests Required

- Creating resonance twice keeps count at `1`.
- Deleting resonance returns count to `0`.
- Echo creation appears in the voice echo list.
- Closed-echo voice rejects echo creation with `CONFLICT`.
- `/api/me` returns interaction summary for the current frequency.

### 7. Wrong vs Correct

#### Wrong

```typescript
await db.update(voices).set({ resonanceCount: voices.resonanceCount + 1 });
```

#### Correct

```typescript
await createVoiceResonance(voiceId, frequency.id);
```

## Scenario: Bottles, Moderation, and AI Assistance

### 1. Scope / Trigger

- Trigger: bottle summary/open/echo/keep flow, public report intake, admin moderation, and AI assistance endpoints.
- Applies to Web now and Android Kotlin client later.
- Admin and AI are server-side only. The client never calls providers directly.

### 2. Signatures

API:

```text
GET    /api/bottles/summary
POST   /api/bottles
POST   /api/bottles/open
POST   /api/bottles/:id/echoes
POST   /api/bottles/:id/continue
POST   /api/bottles/:id/keep
POST   /api/reports
GET    /api/admin/overview
GET    /api/admin/voices
PATCH  /api/admin/voices/:id/status
GET    /api/admin/reports
PATCH  /api/admin/reports/:id
GET    /api/admin/users
PATCH  /api/admin/users/:id/status
POST   /api/ai/classify
POST   /api/ai/tags
POST   /api/ai/moderate
POST   /api/ai/echo-suggestion
```

Database:

```text
bottles(id, author_id, content, allow_echo, status, opened_at, opened_by_id, kept_at, kept_by_id, reply_count, created_at, updated_at)
bottle_echoes(id, bottle_id, author_id, content, status, created_at, updated_at)
reports(id, target_type, target_id, reporter_id, reason, note, status, resolved_by_id, resolution, created_at, updated_at)
admin_actions(id, admin_id, target_type, target_id, action, details, created_at)
ai_reviews(id, review_type, input_text, output_json, provider, model, status, target_type, target_id, created_at)
daily_limits(id, user_id, limit_type, limit_date, count, updated_at)
```

Provider adapter:

```text
POST <AI_BASE_URL>/chat/completions
```

Chat model routing:

```text
fast profile    -> classify, tags, moderate, content-analysis, signal intent/review/quality
default profile -> translate, signal expansion, signal content, fingerprint, program, echo
```

When `AI_BASE_URL` is empty, the adapter must not call the network. It uses local deterministic heuristics instead.

### 3. Contracts

- `GET /api/bottles/summary` may return `remainingBottleCount` and `remainingOpenCount` as `null` when no frequency is attached.
- Bottle echo is valid only when the bottle is opened by the current frequency, `allow_echo=true`, and status is `RECEIVED`.
- Bottle continue and keep are valid only when the bottle is opened by the current frequency and status is `RECEIVED` or `REPLIED`.
- `PATCH /api/admin/reports/:id` accepts only `RESOLVED` or `REJECTED`.
- AI endpoints return a `reviewId` plus a structured JSON result.
- `AI_BASE_URL=""` is valid for local development and demos. AI endpoints still return structured results through local heuristics.
- `AI_MODEL` defaults to `local-fallback`; `provider` and `model` from env are written to `ai_reviews`.
- Optional `AI_FAST_*` env vars define the fast chat profile. If `AI_FAST_API_KEY` is empty, fast env config reuses `AI_API_KEY`.
- Database `chat` provider rows may set `extraJson.chatProfile` to `fast` or `default`; missing or invalid values mean `default`.
- Admin mutations write `admin_actions`.
- AI calls write `ai_reviews`.

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Missing session for bottle/report/admin mutation | 401 | UNAUTHORIZED |
| Non-admin uses admin API | 403 | FORBIDDEN |
| Bottle daily create/open limit reached | 429 | RATE_LIMITED |
| Bottle state does not allow echo/keep/continue | 409 | CONFLICT |
| Bottle was not opened by current frequency | 404 | NOT_FOUND |
| Invalid AI payload | 400 | VALIDATION_ERROR |
| `AI_BASE_URL` is empty | 200 | local fallback |
| AI provider configured but unavailable | 503 | AI_UNAVAILABLE |
| Invalid report/admin status payload | 400 | VALIDATION_ERROR |

### 5. Good/Base/Bad Cases

- Good: separate voice, user, and report status schemas per admin route.
- Base: AI routes store review rows and return `reviewId`.
- Base: empty `AI_BASE_URL` uses local heuristics and still stores the `ai_reviews` row.
- Bad: skip `admin_actions` or leak raw provider payloads to the client.

### 6. Tests Required

- Bottle summary, create, open, echo, continue, and keep route tests.
- Bottle echo service test for invalid state mapping to `CONFLICT`.
- Public report create plus admin report resolve tests.
- Admin permission tests for 401 and 403.
- AI route tests for structured JSON and validation failures.
- AI provider tests for empty-base-url fallback and configured-provider `AI_UNAVAILABLE`.
- AI provider tests for fast/default chat profile routing and timeout budgets.

### 7. Wrong vs Correct

#### Wrong

```typescript
// One shared status query schema for every admin route.
```

#### Correct

```typescript
// Separate status schemas for voices, users, and reports.
```

## Scenario: AI-Assisted Voice Creation

### 1. Scope / Trigger

- Trigger: `POST /api/voices` creates a voice and uses AI to classify channel, suggest mood tags, and moderate risk.
- Applies to Web now and Android Kotlin client later through the same Fastify API.
- Direct AI endpoints and product creation endpoints have different failure behavior.

### 2. Signatures

API:

```text
POST /api/voices
GET  /api/voices/:id
GET  /api/channels/:channel/voices
POST /api/ai/classify
POST /api/ai/tags
POST /api/ai/moderate
```

Service:

```text
classifyContent(body) -> throws AI_UNAVAILABLE on provider failure
classifyContentWithFallback(body) -> returns structured fallback
suggestTagsWithFallback(body) -> returns structured fallback
moderateContentWithFallback(body) -> returns structured fallback
```

Database:

```text
voices(id, author_id, channel, content, mood_tags, allow_echo, status, created_at, updated_at)
ai_reviews(id, review_type, input_text, output_json, provider, model, status, target_type, target_id, created_at)
```

### 3. Contracts

- The client still sends `channel`, `content`, `moodTags`, and `allowEcho`.
- The server may replace the submitted `channel` with AI classification.
- AI classification `STARMAP` is not a direct broadcast target; voice creation must fall back to the submitted broadcast channel.
- If `moodTags` is empty, the server may use AI tags.
- If `moodTags` is non-empty, user tags remain authoritative.
- Moderation maps `ALLOW -> PUBLISHED`, `REVIEW -> REVIEWING`, and `HIDE -> HIDDEN`.
- Hidden or reviewing voices are saved. They are not hard-deleted by AI.
- Public list/detail endpoints only expose `PUBLISHED`.
- The author may read their own non-deleted voice detail.

### 4. Validation & Error Matrix

| Condition | Status | Code / Behavior |
|-----------|--------|-----------------|
| Invalid create voice body | 400 | VALIDATION_ERROR |
| Missing session for create | 401 | UNAUTHORIZED |
| Direct AI endpoint provider failure | 503 | AI_UNAVAILABLE |
| Voice creation AI failure | 201 | fallback result, saved voice |
| Moderation action REVIEW | 201 | saved as REVIEWING |
| Moderation action HIDE | 201 | saved as HIDDEN |
| Public reads hidden/reviewing voice | 404 | NOT_FOUND |

### 5. Good/Base/Bad Cases

- Good: `POST /api/voices` calls server-side AI helpers before writing the voice.
- Good: provider failure during voice creation records fallback AI review rows and keeps the user content.
- Base: direct `/api/ai/*` endpoints return `AI_UNAVAILABLE` when a configured provider fails.
- Bad: frontend calls the model directly or deletes content when AI marks it risky.

### 6. Tests Required

- AI service test: direct AI call returns `AI_UNAVAILABLE` for invalid provider output.
- AI service test: business fallback call records `status=FALLBACK`.
- Voice route test: server-side AI classification and tags affect created voice.
- Voice route test: `HIDDEN` voices are saved, visible to author, and hidden from public list/detail.

### 7. Wrong vs Correct

#### Wrong

```typescript
const moderation = await moderateContent({ content });
if (moderation.result.action === "HIDE") {
  await deleteVoice(id);
}
```

#### Correct

```typescript
const moderation = await moderateContentWithFallback({ content });
const status =
  moderation.result.action === "ALLOW"
    ? "PUBLISHED"
    : moderation.result.action === "HIDE"
      ? "HIDDEN"
      : "REVIEWING";
await createVoiceWithDailyLimit({ ...input, status }, maxDailyVoiceCount);
```

## Scenario: AI Echo Suggestion API

### 1. Scope / Trigger

- Trigger: Web and Android Kotlin client need an AI-generated echo draft for voice and bottle reply flows.
- The endpoint generates an editable draft only. It must never publish the echo for the frequency.
- AI calls stay server-side through the `modules/ai` adapter.

### 2. Signatures

API:

```text
POST /api/ai/echo-suggestion
```

### 3. Contracts

Request:

```json
{
  "content": "source voice or bottle content"
}
```

Constraints:

```text
content: trimmed string, 1..1000 chars
session: required
frequency.status: ACTIVE only
```

Response:

```json
{
  "data": {
    "reviewId": "uuid",
    "suggestion": "editable echo draft"
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

### 4. Validation & Error Matrix

| Condition | Status | Code |
|-----------|--------|------|
| Missing session | 401 | UNAUTHORIZED |
| Frequency is muted or banned | 403 | FORBIDDEN |
| Invalid content body | 400 | VALIDATION_ERROR |
| AI provider unavailable | 503 | AI_UNAVAILABLE |

### 5. Good/Base/Bad Cases

- Good: UI sends source content to `POST /api/ai/echo-suggestion`, fills the reply textarea, then the frequency edits and submits through the normal echo API.
- Base: The response creates an `ai_reviews` record with `reviewType='echo-suggestion'`.
- Bad: Frontend calls the AI provider directly or auto-submits the generated suggestion.

### 6. Tests Required

- Route rejects missing session with `UNAUTHORIZED`.
- Route rejects non-ACTIVE frequency with `FORBIDDEN`.
- Route returns `{ reviewId, suggestion }` for ACTIVE frequency.
- Route still validates request body with `VALIDATION_ERROR`.

### 7. Wrong vs Correct

#### Wrong

```typescript
await fetch(process.env.AI_BASE_URL!, { method: "POST" });
await apiClient.post(`/api/voices/${voiceId}/echoes`, { content: suggestion });
```

#### Correct

```typescript
const { suggestion } = await apiClient.post("/api/ai/echo-suggestion", { content });
setEchoDraft(suggestion);
```

## Scenario: AI Evaluation Evidence API

### 1. Scope / Trigger

- Trigger: coursework/demo-facing AI features need reproducible evidence, not only product behavior.
- Applies to AI classification, emotion/tag extraction, moderation, and bottle matching evaluation.
- Evaluation data must be fixed fixtures or synthetic samples. Do not use private user text as the default evidence source.
- The API is admin/internal. It must still use the shared JSON envelope so Web and future clients can consume the same contract.

### 2. Signatures

API:

```text
GET /api/admin/ai-evaluation
```

Service signatures:

```text
readAiEvaluation(frequency) -> AiEvaluationDashboard
calculateClassificationMetrics(samples, predictions) -> AiMetricGroup
calculateTagMetrics(samples, predictions) -> AiMetricGroup
calculateModerationMetrics(samples, predictions) -> AiMetricGroup
calculateMatchingMetrics(samples, results) -> AiMetricGroup
```

Optional fixture module:

```text
apps/api/src/modules/ai-evaluation/fixtures.ts
```

### 3. Contracts

Response:

```json
{
  "data": {
    "evaluation": {
      "generatedAt": "2026-05-13T00:00:00.000Z",
      "sampleCount": 100,
      "metricGroups": [
        {
          "id": "classification",
          "title": "频道分类",
          "metrics": [
            {
              "label": "Accuracy",
              "value": 0.82,
              "unit": "ratio"
            }
          ]
        }
      ],
      "baselines": [
        {
          "id": "random",
          "title": "随机基线",
          "scores": {
            "classificationAccuracy": 0.33,
            "matchTopKRelevance": 0.21
          }
        }
      ],
      "ablations": [
        {
          "id": "without-emotion",
          "title": "去掉情绪兼容",
          "scores": {
            "matchTopKRelevance": 0.58
          }
        }
      ]
    }
  },
  "meta": {
    "requestId": "req_123"
  }
}
```

Required metric ids:

```text
classificationAccuracy
tagTopKHitRate
moderationRiskRecall
matchTopKRelevance
```

Required baseline ids:

```text
random
rule
final
```

Required ablation ids should match the implemented scorer inputs, for example:

```text
without-semantic
without-emotion
without-freshness
```

### 4. Validation & Error Matrix

| Condition | Status | Code / Behavior |
|-----------|--------|-----------------|
| Missing session | 401 | UNAUTHORIZED |
| Non-admin frequency | 403 | FORBIDDEN |
| Empty fixture set | 200 | return empty metric groups and `sampleCount=0` |
| Invalid fixture shape in code | 500 | INTERNAL_ERROR |
| AI provider unavailable | 200 | use fallback/precomputed evaluation path |

### 5. Good/Base/Bad Cases

- Good: metric calculators are pure functions fed by fixed samples and predictions.
- Good: UI receives compact metric groups and can render poster-ready result blocks without recomputing metrics.
- Base: local demos with no `AI_BASE_URL` still return deterministic fallback evaluation data.
- Bad: evaluation endpoint scans arbitrary production `voices.content` and exposes private text in the response.

### 6. Tests Required

- Route test: missing session returns `UNAUTHORIZED`.
- Route test: non-admin frequency returns `FORBIDDEN`.
- Route test: admin response uses `{ data: { evaluation }, meta }`.
- Unit test: classification accuracy handles zero samples and normal samples.
- Unit test: tag Top-K hit rate counts one expected-tag hit per sample.
- Unit test: moderation risk recall uses risky samples as denominator.
- Unit test: matching Top-K relevance compares selected bottle ids against expected ids.
- Regression test: response does not include raw private user content.

### 7. Wrong vs Correct

#### Wrong

```typescript
const rows = await db.select().from(voices);
return sendData(reply, request, {
  rows: rows.map((row) => ({ content: row.content, ai: row.aiAnalysis })),
});
```

#### Correct

```typescript
const evaluation = readAiEvaluation(request.requireFrequency());
return sendData(reply, request, { evaluation });
```

## Scenario: Admin Signal Pagination API

### 1. Scope / Trigger

- Trigger: changing admin signal lists, seed library lists, moderation search,
  or admin dashboard data contracts.
- Applies to backend API, shared contracts, and Web admin UI because the admin
  panel must browse all records without loading the full table at once.

### 2. Signatures

```text
GET /api/admin/signals
Query:
  limit?: 1..100
  cursor?: opaque string
  q?: 1..200 chars
  source?: human | seed | ai_generated
  status?: DRAFT | PUBLISHED | REVIEWING | HIDDEN | DELETED
Success: 200 { signals: AdminSignalPage }

GET /api/admin/ai-signals
Query:
  limit?: 1..100
  cursor?: opaque string
  q?: 1..200 chars
  source?: human | seed | ai_generated
Success: 200 { signals: SignalDetailPage }

AdminSignalPage:
  items: AdminSignalRecord[]
  total: number
  limit: number
  hasNextPage: boolean
  nextCursor?: string
```

### 3. Contracts

- Admin list endpoints must not return unbounded arrays.
- Default page size is 50; maximum accepted page size is 100.
- `cursor` is opaque to clients and encodes the previous page's last
  `(createdAt, id)` boundary.
- Sort order is newest first: `createdAt DESC, id DESC`.
- Cursor filtering must use the same sort fields:
  `createdAt < cursor.createdAt OR (createdAt = cursor.createdAt AND id < cursor.id)`.
- `total` counts all records matching `source/status/q`, not only the current page.
- `q` search belongs in SQL, not in React state. It should cover frequency code,
  seed text, expanded text, status, and source.
- The admin dashboard may locally paginate already-small lists such as reports
  or AI review records, but signal and seed-library browsing must use server
  pagination.

### 4. Validation & Error Matrix

| Condition | Status | Code / Behavior |
|-----------|--------|-----------------|
| Missing session | 401 | UNAUTHORIZED |
| Non-admin frequency | 403 | FORBIDDEN |
| Invalid `limit` | 400 | VALIDATION_ERROR |
| Invalid `source/status` | 400 | VALIDATION_ERROR |
| Empty result | 200 | `items=[]`, `total=0`, `hasNextPage=false` |
| Malformed cursor | 200 | treat as first page |

### 5. Good/Base/Bad Cases

- Good: admin opens signal list and receives the newest 50 rows plus a cursor.
- Good: admin searches a seed phrase and the API searches the database, not only
  the already-loaded page.
- Good: admin clicks next page and Web sends the opaque cursor back to the API.
- Base: old dashboards with fewer than 50 rows still show one page.
- Bad: `GET /api/admin/signals` returns every signal row.
- Bad: Web filters only the first 50 rows and labels it as all records.

### 6. Tests Required

- Route test: query parameters are validated and passed to admin signal service.
- Service/repository test: list query fetches `limit + 1` rows and returns
  `hasNextPage` plus `nextCursor`.
- Type check: Web and API both consume shared `AdminSignalPage` /
  `SignalDetailPage` contracts.
- Web check: admin signal and seed-library tables call API on page/search changes.

### 7. Wrong vs Correct

#### Wrong

```typescript
const rows = await listAdminSignals({ limit: 50 });
return sendData(reply, request, { signals: rows });
```

#### Correct

```typescript
const signals = await readAdminSignals(frequency, query);
return sendData(reply, request, { signals });
```
