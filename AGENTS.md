# AGENTS.md

## Status

Pre-alpha **design repository**. There is no CLI, `package.json`, test suite, or CI yet.
Do not assume commands exist. Read the design docs before writing code.

Read first:

1. `README.md`
2. `docs/PLAN.md`
3. `docs/ARCHITECTURE.md`
4. `examples/l10n.yml`

## Goal

Build a config-driven tool and GitHub Action that extracts **UI-only** strings from any
GitHub repository, translates them, applies changes safely, verifies the build, and opens
a reviewable PR.

Human review is the default. Auto-release is opt-in.

## First task: Phase 0

1. Create the Node.js package and CLI skeleton.
2. Define the candidate, config, TM, and adapter schemas from `docs/ARCHITECTURE.md`.
3. Port the generic parts of the Viap localization pipeline: TM, validators, apply,
   providers. Leave Viap-specific pins, denylist, and fork patches behind.
4. Add Viap as the first regression fixture.

Exit criteria: Viap's existing localization suite passes through the new core with no
behavior regression.

## Non-negotiable rules

- UI strings only. Do not translate docs, logs, tests, fixtures, or backend messages.
- Only `high` confidence strings auto-apply. `medium` goes to review; `low` is skipped.
- Never auto-merge. Output is a PR.
- Adapters use parsers; no regex-only extractors for supported languages.
- Every adapter needs fixtures, extraction tests, and apply tests.
- No secret or provider key access in code that runs untrusted repository scripts.
- Send only UI strings to providers, never full source or repository paths.
- Verify: parse, placeholders, tags, diff scope, idempotency, then build/tests.
- Add dependencies only when they clearly earn their place.

## Layout

```text
docs/          design and roadmap
examples/      sample configuration
packages/      core, CLI, adapters, action (to be created)
fixtures/      sample repositories per adapter (to be created)
```

## Commits

Short conventional prefixes: `feat:`, `fix:`, `docs:`, `test:`, `ci:`.

## Source of truth

`docs/ARCHITECTURE.md` defines the data model and adapter contract. `docs/PLAN.md` defines
scope and phase exit criteria. If implementation and docs disagree, update the docs in the
same PR or stop and ask.
