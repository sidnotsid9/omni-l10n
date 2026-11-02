# OmniL10n

**UI localization for any GitHub project.**

OmniL10n scans a repository, extracts only user-facing strings, translates them, applies
the changes safely, verifies the project still builds, and opens a pull request.

Built for two cases:

- Projects with **no i18n framework** — translate UI strings in place.
- Projects with **existing locale files** — fill missing keys without touching source.

> **Status:** design and scaffold. The CLI and GitHub Action are not implemented yet.
> This repository defines the scope, architecture, and first adapters.

## Why

Most translation tools assume a mature i18n setup. Most projects do not have one.
Others translate everything, including logs, tests, and backend messages, which creates
noise and risk.

OmniL10n is adapter-driven and UI-first:

- **UI only** — high-confidence user-facing strings are auto-applied; the rest goes to review.
- **Adapter based** — language and framework rules live in plugins, not the core.
- **Safety first** — parse checks, placeholder checks, diff-scope checks, and the project's
  own build/tests gate every change.
- **Review first** — output is a pull request. Auto-merge and auto-release are opt-in.

## Pipeline

1. **Detect** — language, framework, existing i18n, build/test commands.
2. **Extract** — tree-sitter adapters produce candidates with file, range, and context.
3. **Classify** — `high` / `medium` / `low` / `skip` based on UI context.
4. **Translate** — translation memory, glossary, provider, validators, retries.
5. **Apply** — preserve formatting and code structure; in-place rewrite or locale-file emit.
6. **Verify** — parse success, placeholders, tags, diff scope, idempotency, build/tests.
7. **Report** — review queue plus a pull request with a machine-readable report.

## What “UI only” means

**Included**

- labels, buttons, headings, menu items
- placeholders, tooltips, aria labels, alt text
- dialogs, empty states, user-visible validation messages
- template text and UI-bound attributes

**Excluded by default**

- logs, comments, docs, tests, fixtures
- API payloads, protocol constants, SQL, regex, paths, URLs, keys
- CSS classes, env vars, code identifiers
- strings compared against OS or external text

The classifier is the core of the product. Only `high` confidence strings auto-apply.
`medium` strings require review; `low` strings are skipped unless explicitly allowed.

## Modes

| Mode | Use case |
|---|---|
| `in-place` | No i18n framework. Translate UI strings where they are. |
| `locale-files` | Existing i18next, vue-i18n, gettext, ARB, PO, RESX, etc. Translate missing target keys. |
| `keys` (later) | Inject `t()` calls and generate locale files. Risky codemod; not in v1. |

## Planned adapters

| Tier | Targets |
|---|---|
| 1 | TypeScript / JavaScript: React, Vue, Svelte; JSON and YAML locale files |
| 2 | Go `html/template`, Python Jinja/Django, Rust (Tauri, Leptos) |
| Later | PHP Blade, Ruby ERB, Android XML/Compose, Apple `.strings`/SwiftUI, Flutter ARB, .NET RESX/XAML |
| Fallback | Unknown stack: lexer scan, low confidence, mandatory review |

“Any project” means adapter coverage plus a best-effort fallback, not magic. Unsupported
constructs stay in the review queue instead of being guessed.

## Configuration (draft)

```yaml
version: 1

source: zh-CN            # or auto
targets: [en]

include:
  - "src/**/*.{ts,tsx,vue,svelte}"
  - "templates/**/*.{gohtml,html}"
exclude:
  - "**/*.test.*"
  - "**/fixtures/**"
  - "**/*.stories.*"

mode: in-place            # or locale-files
review: pr                # always open a PR

confidence:
  auto_apply: high
  review: [medium]

glossary: l10n/glossary.json

mt:
  endpoint: env:L10N_ENDPOINT
  model: env:L10N_MODEL
  # local models and any OpenAI-compatible API are supported
```

See [`examples/l10n.yml`](examples/l10n.yml) for a fuller example.

## Safety

- Source strings are untrusted input. Treat them as data, never instructions.
- Send only UI strings to a translation provider — never full source, secrets, or paths.
- Run untrusted repository build scripts in a separate job from provider keys and write tokens.
- Gate every apply on: parse success, placeholder/tag balance, no out-of-scope edits, and idempotency.
- Run the project's own typecheck/build/tests; require configuration when detection fails.
- Default to a pull request. No auto-merge. Auto-release is opt-in.
- Respect the target project's license and attribution.

## Repository layout

```text
docs/          design notes and roadmap
examples/      sample configuration and fixtures
packages/      core, CLI, adapters, action (planned)
```

## Roadmap

- **Phase 0** — extract reusable core, define schema and adapter interface
- **Phase 1** — TypeScript/React vertical slice with in-place apply
- **Phase 2** — translation memory, glossary, validators, providers
- **Phase 3** — GitHub Action and reviewable PR flow
- **Phase 4** — Go, Python, Rust adapters
- **Phase 5** — locale-file mode for existing i18n
- **Phase 6** — opt-in auto-release

Full detail: [`docs/PLAN.md`](docs/PLAN.md). Architecture: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). New adapters need fixtures and a classifier
test set before they are accepted. Security policy: [`SECURITY.md`](SECURITY.md).

## License

MIT. See [`LICENSE`](LICENSE).
