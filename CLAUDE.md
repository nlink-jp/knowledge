# CLAUDE.md — knowledge

See [`AGENTS.md`](AGENTS.md) for the full picture.

Key rules:
- This repository is **documentation only** — no build, no tests, no releases.
- Japanese (`docs/ja/`) is the authoring source; keep `docs/en/` in sync in the
  same commit. Keep both README catalogs in sync too.
- Every entry follows **symptom → why → how to apply**; origins are cited in
  generalized form (project + rough date, no personal names).
- **Sanitization gate**: never commit environment-specific values — no GCP
  project IDs, service-account emails, tokens, hostnames, absolute local paths,
  or personal names (ADR-015 §5).
- Small, typed commits (`docs:`, `chore:`).
