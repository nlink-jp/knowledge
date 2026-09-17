# knowledge

The nlink-jp engineering knowledge base — lessons learned across the organization's
projects, compiled into themed documents. Each entry follows a
**symptom → why → how to apply** structure and cites its origin in generalized form.

日本語版は [README.ja.md](README.ja.md) を参照。

## How to use

- **Consult before building** — when starting design or implementation work in a
  domain covered here, read the relevant document first.
- **Feed back what you learn** — when work surfaces new reusable engineering
  knowledge, contribute it here as part of completing that work.

Both practices are organization policy — see
[CONVENTIONS.md](https://github.com/nlink-jp/.github/blob/main/CONVENTIONS.md)
(§Consult and feed the knowledge base) and
[ADR-015](https://github.com/nlink-jp/.github/blob/main/adr/015-knowledge-repository.md).

Documents are consumed by reading `main` — there are no releases.

## Catalog

| Document | Contents |
|---|---|
| [release-engineering](docs/en/release-engineering.md) | Signing, notarization, Homebrew tap, release archives, versioning, licence notices |
| [macos-gui](docs/en/macos-gui.md) | SwiftUI/AppKit traps, menu-bar apps, macOS 27 feasibility and spacing validation, the linked SDK deciding an app's appearance, Wails |
| [web-ui](docs/en/web-ui.md) | CSS/layout traps in WebView frontends and self-contained HTML reports |
| [mcp-server-design](docs/en/mcp-server-design.md) | MCP protocol limits, OAuth discovery limits, stdio hygiene, structured errors, LLM-facing tool design, offset paging with a document id, per-call file output roots |
| [llm-integration](docs/en/llm-integration.md) | Gemini/genai SDK, output validation, drift, tokens, dedup, pipelines, agent tool exclusion, choosing a judge model by generation not thinking level, a local server's prefix cache vs the system prompt, retry bounds from replayed requests, where a local model acts on standing directives (first user message, not the system prompt) |
| [security](docs/en/security.md) | Prompt-injection defense, secrets/PII hygiene, internet-facing checklist, destructive-op safety, quarantining unwrappable tool descriptions, toolchain caches inside a sandbox, moving a judgment to the kernel where one exists and deleting the rule where none does, SSRF closed inside the dialer |
| [build-and-packaging](docs/en/build-and-packaging.md) | CGO cross-builds, .gitignore traps, CI-less release rationale |
| [testing](docs/en/testing.md) | Real-data E2E, delivery verification, mockability, failure injection, MCP test harnesses, cross-platform verification, error-message quality, evidence quality when diagnosing, log timestamp semantics, convergence of independent verification passes, driving the real artifact through a new boundary, model-comparison bench design, measuring inline TUI rows in tmux, multi-row output erasing below itself, terminal capability probes, compiling the prose a model reads, interval cuts at a boundary a segment only touches, view-draw counters in headless tests, single-client fixtures and parallel suites, pinning a defect you are not fixing |
| [containers-and-infra](docs/en/containers-and-infra.md) | Podman on macOS, DuckDB bind mounts, matplotlib fonts, log rotation, SSH liveness checks, why case-sensitive volumes break things, a real guest OS with QEMU/TCG under Podman, matching a VM's devices to the product |
| [config-and-io](docs/en/config-and-io.md) | Bubble Tea's Send-from-Update freeze, Canonical identifiers, strict config decode, data retention expiry, storage-dir reconcile, volume free space, OAuth, terminal IO, send windows held open by an optional reply |
| [shell-scripting](docs/en/shell-scripting.md) | BSD/GNU sed differences, zsh expansion quirks, Bash trap scope, substitution pitfalls |
| [embedded](docs/en/embedded.md) | M5Stack / ESP32 lessons |
| [development-process](docs/en/development-process.md) | Rewrite-vs-refactor, contribution triage, ADR granularity, docs practice, separating archived repositories, broken submodules, reports vs controls, ADR citations in ported code, judging whether an upstream proposal is wanted |

Japanese versions live in [docs/ja/](docs/ja/) (Japanese is the authoring source).

## License

[MIT](LICENSE)
