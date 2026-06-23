# Rewrite README into project introduction

## Goal

Turn the root `README.md` into a real project introduction for Unfreq, and move deployment, operations, migration, development, environment, and quality-check material into focused docs under `docs/`.

## What I Already Know

* The current README starts with a concise Unfreq description, then becomes a long Docker deployment and operations guide.
* The user wants README to read like a project introduction instead of deployment steps.
* Existing docs include debug playbooks and project explanation/defense material.
* The repo is a monorepo with `apps/web`, `apps/api`, `packages/contracts`, `deploy`, `scripts`, and `compose.yaml`.

## Requirements

* Rewrite `README.md` as a concise project overview.
* Preserve the product identity: AI 同频匹配的可视化共振社区.
* Explain core loop, product boundaries, architecture, repository layout, and key docs links.
* Move deployment content out of README into dedicated docs.
* Split long operational material into readable focused files instead of one oversized document.
* Preserve existing commands, environment variable notes, migration checklist, backup/restore guidance, and quality checks.
* Borrow the AstrBot docs shape: a docs landing page, quick-start framing, clear category splits, and FAQ-style support pages.
* Keep a detailed architecture and rationale document that explains how each major part is built and why the stack choices were made.

## Acceptance Criteria

* [ ] README reads as a project introduction before any setup or deployment detail.
* [ ] README links to the new operational docs.
* [ ] Docker deployment steps live in a dedicated doc.
* [ ] server migration steps live in a dedicated doc.
* [ ] local development and quality commands live in a dedicated doc.
* [ ] environment and secret notes remain available.
* [ ] no command from the original README is lost unless duplicated by a more precise doc.
* [ ] architecture/rationale docs explain frontend, backend, shared contracts, AI layer, data layer, deployment, and quality tooling.
* [ ] docs home mirrors the AstrBot-style overview with concise category navigation.
* [ ] usage and FAQ pages exist for the main product and common support questions.

## Definition of Done

* Documentation changes are scoped to README and `docs/`.
* Markdown links point to existing files.
* `git diff --check` passes.
* Full build/test is not required unless code changes are introduced.

## Technical Approach

Rewrite the README as a front door:

* what Unfreq is;
* what problem it solves;
* how the main product loop works;
* what the system is made of;
* where to find deployment, operations, migration, development, and debug docs.

Create focused docs:

* `docs/deployment.md` for Docker Compose server deployment and preflight/smoke checks.
* `docs/operations.md` for logs, restart, backup, restore, admin promotion, optional deploy script, and environment variables.
* `docs/server-migration.md` for migration checklist, backup, restore, validation, and rollback.
* `docs/development.md` for local commands, test scope, repo layout, terminology, and quality checks.
* `docs/architecture.md` for system composition, how each part is built, and stack rationale.
* `docs/index.md` for the docs landing page.
* `docs/usage.md` for the user-facing flow.
* `docs/faq.md` for common questions and support answers.

## Decision (ADR-lite)

**Context**: The current README has useful information, but it mixes project introduction with deployment runbook content.

**Decision**: Keep README as a concise project overview and route procedural content to focused docs under `docs/`.

**Consequences**: The root page becomes easier to read, while operational details remain available and easier to maintain.

## Out of Scope

* Product UI changes.
* Deployment script changes.
* New runtime behavior.
* External publishing or GitHub metadata updates.

## Technical Notes

* Current README inspected.
* Existing docs inspected: `docs/debug/index.md`, `docs/project-principles-and-literature-review.md`, `docs/project-implementation-and-defense-guide.md`.
* Scripts confirmed from `package.json` and `scripts/`.
