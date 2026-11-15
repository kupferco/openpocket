# Contributing

## Secrets

OpenPocket is a public repo. Anything committed here is public forever, even if you delete it later.

- Never commit secrets: API keys, tokens, passwords, private keys, service-account files, or personal data.
- Read configuration from environment variables. Document each variable, with a fake value, in `.env.example`.
- `.env` and similar files (`.env.*`, `*.pem`, `*.key`, service-account JSON) are git-ignored. Keep it that way.
- CI runs [gitleaks](https://github.com/gitleaks/gitleaks) on every push and pull request and fails on findings.
- Optional, to catch problems before you commit: `pip install pre-commit && pre-commit install`.
- If you committed a secret by accident, rotate it first, then tell the maintainer. Do not rely on deleting the commit.
