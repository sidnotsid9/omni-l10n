# Architecture

## Overview

OmniL10n is a pipeline, not a single translator.

```text
repository
   |
   v
[ detect ] -- languages, frameworks, i18n, build commands
   |
   v
[ extract ] -- adapter(s) -> candidates
   |
   v
[ classify ] -- high / medium / low / skip
   |
   v
[ translate ] -- TM -> glossary -> provider -> validate -> retry
   |
   v
[ apply ] -- in-place edits or locale-file values
   |
   v
[ verify ] -- parse, placeholders, scope, idempotency, build/tests
   |
   v
[ report ] -- review queue + pull request
```

Each stage writes a deterministic artifact that the next stage can consume. That makes
runs resumable and failures debuggable.

## Artifacts

```text
.omni-l10n/
  scan.json          detected languages, frameworks, commands
  corpus.json        extracted candidates
  review.json        medium/low confidence items
  tm.json            accepted translations for this project
  report.json        verification and diff summary
  report.md          human-readable PR body
```

Only `.l10n.yml`, locale/data files, and source edits are committed. `.omni-l10n/` is local.

## Candidate model

```ts
type Confidence = "high" | "medium" | "low" | "skip";

type Candidate = {
  id: string;              // content hash of source + context + placeholders
  file: string;
  range: { start: number; end: number };
  source: string;
  target?: string;
  kind: string;            // jsx-text, jsx-attr, template-text, locale-value, ...
  context: string;         // component, page, template, locale
  confidence: Confidence;
  placeholders: string[];  // {0}, %s, ${x}, ICU, ...
  markup: string[];        // <b>, <1>, markdown, ...
  notes?: string[];
};
```

Rules:

- A candidate is immutable once extracted. Translation fills `target`.
- The ID changes when the source, context, or placeholder set changes.
- Apply uses `file` + `range`; it never searches globally for a string.

## Adapter contract

```ts
interface Adapter {
  id: string;
  detect(repo: RepoInfo): AdapterScore;
  extract(files: SourceFile[]): Candidate[];
  apply(edits: Edit[]): void;
}
```

Constraints:

- No network access.
- No secret access.
- No project code execution.
- Must return ranges relative to the original file bytes or UTF-16 code units, documented
  explicitly.

Parsers: tree-sitter for structured languages; framework-specific visitors on top. The
fallback adapter is a lexer scan that only produces `low` confidence items.

## Configuration

```yaml
version: 1
source: zh-CN
targets: [en]

include: ["src/**/*.{ts,tsx,vue,svelte}"]
exclude: ["**/*.test.*", "**/fixtures/**"]

mode: in-place          # in-place | locale-files
review: pr

confidence:
  auto_apply: high
  review: [medium]

glossary: l10n/glossary.json
tm: l10n/tm.json

mt:
  endpoint: env:L10N_ENDPOINT
  model: env:L10N_MODEL
  batch_size: 30
  timeout_ms: 60000
```

`source: auto` enables script detection. It must produce a confidence score, not a guess.

## Translation memory

- Key: candidate ID.
- Value: `{ target, provider, model, reviewed, updated }`.
- A reviewed translation always wins over a new model output.
- Glossaries are enforced after generation and before validation.
- Re-runs only translate candidates whose ID is absent from the TM.

## Providers

Any OpenAI-compatible chat-completions endpoint:

- local llama.cpp / LM Studio
- hosted free tiers
- paid APIs

Provider contract:

```ts
interface Provider {
  translate(batch: Candidate[], options: TranslateOptions): Promise<Map<string, string>>;
}
```

The provider gets strings and context hints only. It never gets repository files, paths,
secrets, or build output.

## Apply engine

### In-place

- Replace only the candidate range.
- Preserve quote style, escaping, indentation, and surrounding trivia.
- Reject the edit if the file does not parse afterward.
- Reject the edit if the AST changes outside approved string nodes.
- Re-run extraction after apply; translated strings must no longer be candidates.

### Locale files

- Parse the format, update only target keys.
- Preserve key order and comments where the format supports them.
- Do not add keys that already exist with a non-empty target.
- Reject on duplicate keys or invalid syntax.

## Verification gates

1. **Parse** — every touched file parses.
2. **Placeholders** — identical multiset before and after.
3. **Markup** — balanced tags, preserved components, valid escaping.
4. **Scope** — diff lines map to candidate ranges or approved formatting.
5. **Idempotency** — applying twice yields the same tree.
6. **Build** — detected typecheck/build/test commands pass.
7. **Residue** — source-language strings are gone from high-confidence UI ranges.

A failure at any gate blocks the PR.

## Security model

The target repository is untrusted.

- Jobs are split: extract -> translate -> apply -> verify.
- Provider keys exist only in the translate job.
- Write tokens exist only in the report/PR job.
- Build/test jobs run without provider keys.
- Do not use `pull_request_target` to run untrusted code with secrets.
- Limit network access and command execution in the build job where possible.
- Treat source strings as prompt-injection input. Use a fixed system prompt, JSON output,
  schema validation, and reject any output that looks like code or instructions.

## GitHub Action

Inputs:

```yaml
config: .l10n.yml
targets: en
mode: in-place
dry_run: false
provider_secret: L10N_API_KEY
```

Outputs:

- pull request URL
- report artifact
- review count

Default permissions: `contents: write`, `pull-requests: write`. No issue/admin scopes.

## Testing strategy

- Unit tests per module.
- Fixture repos per adapter: source tree, expected candidates, expected applied tree.
- Golden files for reports.
- Sabotage fixtures that prove each gate fails when it should.
- One end-to-end fixture per mode: in-place, locale-files, dry-run.
