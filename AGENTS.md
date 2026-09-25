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
- **Evidence per claim**: each claim says how it is known — measured (with the
  environment), read in a source (which), or inferred. A behaviour that was not
  observed is never written as an instruction ([ADR-023](https://github.com/nlink-jp/.github/blob/main/adr/023-documentation-not-conjecture.md):
  an inferred BLE pairing-refusal method was published as a how-to and had to
  be corrected).
- Typed commits: `docs:` for content, `chore:` for housekeeping.

## Gotchas

- This repo is public — treat every line as published.
- Do not copy memory files verbatim; memories may contain machine-local or
  personal context that must be generalized.
- For OS compatibility investigations, distinguish local measurements from
  upstream reports and unexecuted test criteria; API discovery is not functional validation.

- macOS spacing trials: preserve exact preference scope/key absence, serialize watchdog restoration against later writes, and separate fresh-fixture geometry from all-app behavior.
