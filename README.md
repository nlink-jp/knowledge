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
| [release-engineering](docs/en/release-engineering.md) | Signing, notarization, Homebrew tap, release archives, versioning |
| [macos-gui](docs/en/macos-gui.md) | SwiftUI/AppKit traps, menu-bar apps, Wails |
| [web-ui](docs/en/web-ui.md) | CSS/layout traps in WebView frontends and self-contained HTML reports |
| [mcp-server-design](docs/en/mcp-server-design.md) | MCP protocol limits, stdio hygiene, structured errors, LLM-facing tool design |
| [llm-integration](docs/en/llm-integration.md) | Gemini/genai SDK, output validation, drift, tokens, dedup, pipelines |
| [security](docs/en/security.md) | Prompt-injection defense, secrets/PII hygiene, internet-facing checklist, destructive-op safety |
| [build-and-packaging](docs/en/build-and-packaging.md) | CGO cross-builds, .gitignore traps, CI-less release rationale |
| [testing](docs/en/testing.md) | Real-data E2E, mockability, failure injection, MCP test harnesses |
| [containers-and-infra](docs/en/containers-and-infra.md) | Podman on macOS, DuckDB bind mounts, matplotlib fonts |
| [config-and-io](docs/en/config-and-io.md) | Canonical identifiers, strict config decode, OAuth, terminal IO |
| [shell-scripting](docs/en/shell-scripting.md) | BSD/GNU sed differences, zsh expansion quirks, Bash trap scope, substitution pitfalls |
| [embedded](docs/en/embedded.md) | M5Stack / ESP32 lessons |
| [development-process](docs/en/development-process.md) | Rewrite-vs-refactor, contribution triage, ADR granularity, docs practice |

Japanese versions live in [docs/ja/](docs/ja/) (Japanese is the authoring source).

## License

[MIT](LICENSE)
