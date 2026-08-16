# knowledge

nlink-jp のエンジニアリングナレッジベース — 組織のプロジェクト横断で得られた知見を
テーマ別ドキュメントに集約したもの。各項目は**事象 → なぜ → 適用方法**の構造を持ち、
出所を一般化した形で記載している。

English version: [README.md](README.md)

## 使い方

- **作る前に読む** — 本書がカバーする領域の設計・実装に着手する前に、該当ドキュメントを
  読む。
- **学んだら還元する** — 作業中に再利用可能な知見が生まれたら、その作業の完了の一部として
  ここへ還元する。

いずれも組織ポリシー —
[CONVENTIONS.md](https://github.com/nlink-jp/.github/blob/main/CONVENTIONS.md)
（§Consult and feed the knowledge base）と
[ADR-015](https://github.com/nlink-jp/.github/blob/main/adr/015-knowledge-repository.md)
を参照。

ドキュメントは `main` を読んで利用する（リリースは無い）。

## カタログ

| ドキュメント | 内容 |
|---|---|
| [release-engineering](docs/ja/release-engineering.md) | 署名・notarization・Homebrew tap・リリースアーカイブ・版数 |
| [macos-gui](docs/ja/macos-gui.md) | SwiftUI/AppKit の罠、メニューバーアプリ、Wails |
| [web-ui](docs/ja/web-ui.md) | WebView フロントエンドと自己完結 HTML レポートの CSS/レイアウトの罠 |
| [mcp-server-design](docs/ja/mcp-server-design.md) | MCP プロトコルの制約、stdio 衛生、構造化エラー、LLM 向けツール設計 |
| [llm-integration](docs/ja/llm-integration.md) | Gemini/genai SDK、出力検証、drift、トークン、dedup、パイプライン |
| [security](docs/ja/security.md) | プロンプトインジェクション対策、秘密情報/PII、公開サービス、破壊的操作の安全設計 |
| [build-and-packaging](docs/ja/build-and-packaging.md) | CGO クロスビルド、.gitignore の罠、CI 不使用の判断理由 |
| [testing](docs/ja/testing.md) | 実データ E2E、モック設計、失敗注入、MCP テストハーネス、診断時の証拠の質 |
| [containers-and-infra](docs/ja/containers-and-infra.md) | macOS の Podman、DuckDB bind mount、matplotlib フォント |
| [config-and-io](docs/ja/config-and-io.md) | canonical 識別子、strict 設定デコード、保存先変更と reconcile、ボリューム空き容量、OAuth、ターミナル IO |
| [shell-scripting](docs/ja/shell-scripting.md) | BSD/GNU sed 差、zsh 展開の癖、Bash trap スコープ、置換の罠 |
| [embedded](docs/ja/embedded.md) | M5Stack / ESP32 の知見 |
| [development-process](docs/ja/development-process.md) | rewrite/refactor 判断、コントリビューション triage、ADR 粒度、ドキュメント作法 |

英語版は [docs/en/](docs/en/)（日本語が原文）。

## License

[MIT](LICENSE)
