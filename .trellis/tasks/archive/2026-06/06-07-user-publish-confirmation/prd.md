# Add User Publish Confirmation

## Goal

After a user emits a seed, AI generation should finish into a private owner preview instead of immediately publishing the signal. The signal should enter public surfaces only after the owner explicitly confirms publish.

## What I already know

* Current homepage flow calls `POST /api/signals`, writes a `DRAFT` row with `generationStatus=RECEIVED`, then completes it in the background.
* Current completion writes low-risk signals as `status=PUBLISHED`, so result, galaxy, random, match, share, resonance, and save surfaces can expose the expanded text immediately.
* Existing statuses already include `DRAFT`, `PUBLISHED`, and `REVIEWING`.
* Public reads already filter to `status=PUBLISHED` plus `generationStatus=COMPLETED`.
* Owner reads currently cannot see a completed `DRAFT`; `getSignalDetailForFrequency` returns 404 for completed rows that are not `PUBLISHED` or `REVIEWING`.

## Assumptions

* MVP means "preview then publish", not full editing.
* Users can leave the preview unpublished by navigating away.
* AI high-risk signals should still go to `REVIEWING` and should not show a publish button.
* Internal AI-generated/demo signals that use `createPublishedSignal` can keep publishing directly unless they use the homepage pending flow.

## Requirements

* Homepage-created low/medium-risk signals complete as private preview, not public signal.
* The owner can read the generated preview after completion.
* The result page shows seed, expanded text, and a clear publish action for unpublished owner previews.
* Publishing changes the signal to `PUBLISHED` only for the owner.
* Public/other-frequency access to unpublished previews remains hidden.
* Unpublished previews must not appear in galaxy, random, hall, matches, save, resonance, share card, or other public surfaces.
* High-risk signals remain `REVIEWING` and use the existing review UI.

## Acceptance Criteria

* [ ] `POST /api/signals` still returns a generation detail immediately.
* [ ] Completed homepage signals with low/medium risk have `status=DRAFT` until the owner publishes.
* [ ] Owner result page renders a private preview with a publish button.
* [ ] Publish button calls a typed API helper and moves the signal to `PUBLISHED`.
* [ ] Non-owner and anonymous users cannot read private preview content.
* [ ] Public surfaces exclude private previews.
* [ ] Backend route tests cover create-preview, owner read, non-owner hidden, and publish.
* [ ] Frontend validators/types cover the preview response shape and publish response.
* [ ] Relevant lint/type/test commands pass or failures are documented.

## Definition of Done

* Tests added or updated where behavior changes.
* Lint and typecheck run.
* Docs updated if the signal send chain changes.
* No sub-agent usage for this implementation.

## Out of Scope

* Editing expanded text before publish.
* AI rewrite/regenerate from the preview screen.
* Background queue recovery.
* New database enum values or migrations unless unavoidable.
* Full moderation workflow redesign.

## Technical Notes

* Backend creation: `apps/api/src/modules/signals/service-creation.ts`
* Backend reads: `apps/api/src/modules/signals/service-queries.ts`
* Backend routes: `apps/api/src/modules/signals/routes.ts`
* Backend repository: `apps/api/src/modules/signals/repository.ts`
* Contracts: `packages/contracts/src/signals.ts`
* Frontend data: `apps/web/src/lib/signal-data.ts`
* Frontend validators: `apps/web/src/lib/api-response-validators.ts`
* Result page: `apps/web/src/app/result/[id]/page.tsx`
* Existing public filters already require `status=PUBLISHED` and `generationStatus=COMPLETED`.
