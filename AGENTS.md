# AGENTS.md — knowledge

## What this repository is

The nlink-jp engineering knowledge base, established by
[ADR-015](https://github.com/nlink-jp/.github/blob/main/adr/015-knowledge-repository.md).
Themed documents under `docs/{en,ja}/` hold lessons learned across the
organization's projects, compiled from the maintainer's agent memory corpus.
It is a standalone repository — not part of any series umbrella.

## Structure

```
README.md / README.ja.md   # catalog (keep both in sync)
docs/en/<theme>.md         # English documents
docs/ja/<theme>.md         # Japanese documents (authoring source)
CHANGELOG.md
```

## How to edit

- No build, no tests, no releases — documents are consumed by reading `main`.
- Entry format: `### <title>` + **事象/Symptom → なぜ/Why → 適用方法/How to apply**.
- Author in Japanese first, translate to English in the same commit.
- Adding a new theme: create both language files, add a row to both README
  catalogs.
- **Sanitization gate (mandatory before push)**: no environment-specific values —
  GCP project IDs, SA emails, tokens, hostnames, internal IPs, absolute local
  paths, personal names. Use placeholders (`<your-xxx>`, `<TEAM_ID>`).
- Typed commits: `docs:` for content, `chore:` for housekeeping.

## Gotchas

- This repo is public — treat every line as published.
- Do not copy memory files verbatim; memories may contain machine-local or
  personal context that must be generalized.
