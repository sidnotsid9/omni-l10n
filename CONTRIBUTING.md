# Contributing

Thanks for helping build OmniL10n.

## Before you start

- Open an issue for a new adapter, provider, or output mode.
- Keep changes focused. One adapter or one pipeline stage per pull request.
- Never commit secrets, signing keys, private repositories, or generated build output.

## Development

The project is pre-alpha. The intended stack is Node.js 18+ with no runtime dependencies
until a dependency clearly earns its place.

Planned checks:

```bash
npm test
npm run lint
npm run typecheck
```

## Adapter requirements

A new language/framework adapter is accepted only with:

1. A fixture repository or fixture tree under `fixtures/`.
2. Extraction tests covering UI strings, skipped strings, placeholders, and markup.
3. Apply tests proving the source still parses and the diff stays in scope.
4. A short note describing known false positives and false negatives.
5. UI confidence rules documented in the adapter.

## Commit style

Use short conventional prefixes:

```text
feat: add vue adapter
fix: preserve JSX entity escaping
docs: explain locale-file mode
test: cover ICU plural placeholders
```

## Security

- Treat repository content as untrusted.
- Do not run untrusted project scripts in a job that holds provider keys or write tokens.
- Do not send full source files to a translation provider.
- Report security issues privately; do not open a public issue with exploit details.

## License

By contributing, you agree that your contribution is licensed under the MIT License.
