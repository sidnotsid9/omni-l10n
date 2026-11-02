# OmniL10n Plan

## Goal

A config-driven CLI and GitHub Action that:

1. detects a repository's UI surfaces and existing i18n,
2. extracts only user-facing strings,
3. translates them with a translation memory, glossary, and provider,
4. applies changes without breaking code,
5. verifies the project builds and the diff stays in scope,
6. opens a reviewable pull request.

Human review is the default. Auto-release is opt-in.

## Non-goals for v1

- translating docs, comments, logs, tests, or backend-only messages
- native mobile UI (Android/iOS), games, images, audio
- full i18n migration (`t()` key injection)
- auto-merging pull requests
- supporting every language on day one

## Product principles

1. **UI first.** Only high-confidence UI strings auto-apply.
2. **Adapters over regex.** Use parsers and framework rules.
3. **Do not break the build.** A translation that fails verification is not applied.
4. **Review is a feature.** Medium-confidence strings go to a queue.
5. **Small diffs.** Touch only candidate ranges.
6. **No secrets in untrusted code.** Separate extraction, translation, apply, and build jobs.
7. **Best effort is explicit.** Unknown stacks get a low-confidence fallback, not a guess.

## Architecture

```text
CLI / Action
    |
    v
 config -> detect -> extract -> classify -> translate -> apply -> verify -> report/PR
              |          |          |           |          |
              |          |          |           |          +-- in-place rewrite
              |          |          |           |          +-- locale-file emit
              |          |          |           +------------- TM, glossary, provider
              |          |          +------------------------- high / medium / low / skip
              |          +------------------------------------ tree-sitter adapters
              +----------------------------------------------- language/framework/i18n
```

### Core modules

- `config` — parse and validate `.l10n.yml`
- `detect` — languages, frameworks, i18n setup, build/test commands
- `extract` — adapter registry; normalized candidates
- `classify` — UI confidence and exclusion rules
- `translate` — TM lookup, batching, provider, validation, retries
- `apply` — occurrence-based edits; in-place or locale files
- `verify` — parse, placeholders, scope, idempotency, build/tests
- `report` — review queue, metrics, PR body

### Candidate schema

```json
{
  "id": "sha256:...",
  "file": "src/components/Dialog.tsx",
  "range": { "start": 120, "end": 145 },
  "source": "Delete this file?",
  "target": null,
  "kind": "jsx-text",
  "context": "component",
  "confidence": "high",
  "placeholders": [],
  "markup": [],
  "notes": []
}
```

IDs are content hashes of source + context + placeholders. The same text in two contexts
must not collapse into one translation.

### Adapter interface

```js
export default {
  id: "ts-react",
  detect(repo) {},
  extract(files) {},
  apply(edits) {}
};
```

No adapter may read secrets. No adapter may execute project code.

## UI classifier

### Include

- JSX text and UI attributes: `title`, `aria-label`, `placeholder`, `alt`
- template text nodes and UI-bound attributes
- labels, buttons, headings, menu items, empty states
- dialogs, toasts, and user-visible validation messages
- locale values in existing resource files

### Exclude

- tests, fixtures, stories, docs, comments
- logs, internal errors, API payloads, protocol constants
- SQL, regex, paths, URLs, keys, env vars, code identifiers
- CSS, class names, selectors
- strings compared with OS or external output
- strings requiring runtime data that cannot be preserved

### Confidence

| Level | Action |
|---|---|
| `high` | auto-apply after verification |
| `medium` | review queue |
| `low` | skip unless allowlisted |
| `skip` | never translate |

The classifier must be measured on a labeled corpus before an adapter ships.

## Apply modes

### In-place

For projects without i18n. Rewrite the string at its source location, preserving quotes,
escaping, comments, and formatting. Verify the file still parses and the AST differs only
at approved string nodes.

### Locale files

For projects with existing i18n. Detect the framework and file format, add only missing
target keys, preserve ordering and comments where the format allows, and never rewrite
source code.

### Key injection (later)

Generate keys and replace literals with `t()` calls. Requires framework-specific codemods
and a much stronger verification suite. Not part of v1.

## Phases

### Phase 0 — Extract the core

- Create the package and CLI skeleton.
- Define candidate, config, TM, and adapter schemas.
- Port generic pieces from the Viap localization pipeline: TM, validators, apply, providers.
- Use Viap as the first regression fixture.

**Exit:** Viap's existing suite passes through the new core; no behavior regression.

### Phase 1 — TypeScript/React vertical slice

- tree-sitter TS/TSX/JS extraction.
- React JSX text and attribute rules.
- UI confidence scoring and exclusions.
- In-place apply with parse-preserving edits.
- Three small real repositories as fixtures.

**Exit:** 3 PRs opened; all builds green; zero out-of-scope diffs; high-confidence precision
at least 95% on the labeled set.

### Phase 2 — Trust layer

- Translation memory and incremental runs.
- Glossary and project overrides.
- Review queue and report.
- Placeholder, ICU, HTML/Markdown, and format-string validators.
- Local and OpenAI-compatible providers; retries and rate limits.

**Exit:** re-running translates nothing already accepted; provider failure does not corrupt
the tree; cost per 1,000 strings is recorded.

### Phase 3 — GitHub Action and PR flow

- `workflow_dispatch` and optional PR trigger.
- Split jobs: extract, translate, apply, build.
- No provider key or write token in the build job.
- PR body includes summary, review items, and verification results.

**Exit:** installed on 2 repos; a maintainer can review and merge without editing files.

### Phase 4 — More adapters

Priority: Go templates, Python Jinja/Django, Rust/Tauri.

- Go: `html/template`, `text/template`, `.gohtml`; preserve `{{...}}` actions.
- Python: Jinja2 and Django templates; preserve filters and blocks.
- Rust: Tauri frontend (TS) and Leptos/Askama templates where applicable.

**Exit:** one real repository per adapter, each with fixtures and classifier tests.

### Phase 5 — Locale-file mode

- Detect i18next, vue-i18n, gettext/PO, ARB, RESX, Android XML, Apple `.strings`.
- Translate missing keys only.
- Preserve comments and ordering where possible.

**Exit:** works on i18next, gettext, and ARB fixtures without source changes.

### Phase 6 — Opt-in auto-release

- Reuse the Viap-style pipeline: translate, gate, build, release.
- Require signing keys and a release policy.
- Only after a repository has N green PRs.

**Exit:** one opted-in repo updates automatically; failures open an issue and publish nothing.

## Metrics

- extraction precision and recall on labeled repos
- high-confidence precision
- build/typecheck pass rate after apply
- review acceptance rate
- diff-scope violations (must be zero)
- translation cost per 1,000 strings
- time from scan to PR

## Risks

| Risk | Mitigation |
|---|---|
| UI classifier false positives | confidence levels, review queue, labeled fixtures |
| Code breakage | parse checks, build/tests, scope checks, idempotency |
| Prompt injection in source strings | treat as data, strict output schema, validation |
| Secrets exfiltration from untrusted repos | separate jobs, least privilege, no secrets in build |
| Translation quality | glossary, context, TM, human review |
| Adapter sprawl | adapter interface, fixtures, tiered support |
| License/attribution | respect target project licenses; keep attribution |

## First milestone

Build one vertical slice, not ten adapters:

1. a small React/TS repository,
2. a Go `html/template` repository,
3. a repository with existing locale files.

Start with the React slice, then extract the core into reusable modules.
