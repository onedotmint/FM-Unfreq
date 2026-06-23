# Harden public signal visibility and review core UX

## Goal

Strengthen public signal visibility so incomplete AI-generation rows cannot leak through public surfaces, then decide whether the current seed-to-result UX needs one small MVP addition.

## What I Already Know

* The user accepted adding a public-query safety condition.
* The user confirmed expansion provider failure should remain "failed, user can retry"; no local prose fallback is required for this task.
* Current normal creation flow writes `DRAFT + RECEIVED`, then completes the same row as `PUBLISHED/REVIEWING + COMPLETED`.
* Public surfaces should only show published signals that completed generation.
* Current user-facing flow is already broad: seed entry, transmitting/progress page, result page, same-frequency matches, resonance/save/playback/share, galaxy, my frequency, onboarding path, and failed-generation retry.

## Assumptions

* Backend hardening should be narrow and not change normal successful generation behavior.
* UX additions, if any, should be small and directly support the core loop, not open a new product surface.
* "Failed can retry" remains intended behavior.

## Open Questions

* None.

## Requirements

* Public signal reads must require both `status = PUBLISHED` and `generationStatus = COMPLETED`.
* Owner-only generation reads must keep working for non-completed rows.
* Reviewing visibility behavior must not regress.
* Existing retry behavior for failed generation must remain unchanged.

## Acceptance Criteria

* [x] Hall, random, galaxy, detail, matches, resonance/save/audio/echo public reads do not expose `PUBLISHED` rows unless generation is `COMPLETED`.
* [x] Owner can still read their in-progress or failed signal generation detail.
* [x] Reviewing owner and restricted non-owner behavior remains unchanged.
* [x] Tests cover at least one abnormal `PUBLISHED + non-COMPLETED` row on a public surface.
* [x] No UX files or copy are changed.

## Definition of Done

* Tests added or updated for backend visibility hardening.
* Lint/typecheck or focused test commands run.
* Docs/spec updated only if behavior contract changes.
* No local AI expansion fallback added.

## Out of Scope

* Replacing process-local generation with a durable worker.
* Adding local fallback prose for failed expansion.
* Building a new chat surface or multi-turn AI conversation UI.
* Redesigning galaxy or result page visuals.
* Result-page UX polish or copy changes.

## Technical Notes

* `apps/api/src/modules/signals/service-creation.ts` owns pending creation and completion.
* `apps/api/src/modules/signals/service-queries.ts` owns public/detail query behavior.
* `apps/api/src/modules/signals/repository.ts` owns published signal filters.
* `apps/api/src/modules/signals/routes.test.ts` and `service.test.ts` already cover creation, owner generation reads, reviewing access, matching, and public routes.
* `docs/debug/signal-send-chain.md` documents the current state flow and known process-local generation limitation.

## UX Review Notes

* The core loop is already represented: seed entry -> transmitting progress -> result -> same-frequency matches -> resonance/save/playback/share -> galaxy -> my frequency.
* The result page already has a follow-up seed path through `followUpQuestion`.
* The strongest possible gap is not a missing page, but clarity after the result: helping the user understand why a match is "same frequency" or why they should continue with the next seed.
* User chose option 1 on 2026-06-06: only backend hardening for this task. UX work stays out of scope.

## Decision

**Context**: Public queries mostly rely on `status = PUBLISHED`. Normal creation keeps pending rows as `DRAFT`, but abnormal data could combine `PUBLISHED` with a non-terminal generation status.

**Decision**: Harden public visibility by requiring `generationStatus = COMPLETED` wherever a row is treated as publicly published.

**Consequences**: Normal successful signals remain visible. Abnormal or partially generated rows stay hidden from public surfaces. Failed-generation retry behavior remains unchanged.
