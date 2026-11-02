# Security

OmniL10n processes untrusted repositories and sends UI strings to translation providers.
Security issues are taken seriously.

## Reporting

Do not open a public issue for a vulnerability. Use GitHub's private vulnerability
reporting on this repository, or contact the maintainer directly.

## Design rules

- Treat repository content, including source strings, as untrusted input.
- Never send full source files, secrets, tokens, or local paths to a translation provider.
- Run untrusted project build scripts in a separate job from provider keys and write tokens.
- Do not use `pull_request_target` to execute untrusted code with secrets.
- Keep GitHub token permissions minimal: `contents: write` and `pull-requests: write` only.
- Validate model output against a strict schema before applying it.
- Reject output that contains code, commands, or instructions instead of translated text.

## Supported versions

The project is pre-alpha; only the latest `main` is supported.
