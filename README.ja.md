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
| [release-engineering](docs/ja/release-engineering.md) | 署名・notarization・Homebrew tap・リリースアーカイブ・版数・ライセンス表記 |
| [macos-gui](docs/ja/macos-gui.md) | パネルの状態は自分で持つ（isShown は閉じてから遅れる）、SwiftUI/AppKit の罠、メニューバーアプリ、macOS 27の実現性・間隔検証、リンクした SDK が外観を決める、ビューが正しいときはレイヤーの木を吐く、Wails、書き込み時点では CFPreferences も plist も保存の成否を答えない（失敗は設計の前に本物の対象で測る） |
| [web-ui](docs/ja/web-ui.md) | WebView フロントエンドと自己完結 HTML レポートの CSS/レイアウトの罠 |
| [mcp-server-design](docs/ja/mcp-server-design.md) | MCP プロトコルの制約、OAuth discovery の限界、stdio 衛生、構造化エラー、LLM 向けツール設計、文書 id 付き offset ページング、ファイル出力先は呼び出しごとの引数 |
| [llm-integration](docs/ja/llm-integration.md) | Gemini/genai SDK、出力検証、drift、トークン、dedup、パイプライン、エージェントのツール除外、判定モデルは thinking でなく世代で選ぶ、ローカルサーバの接頭辞キャッシュと system プロンプト、再送上限は再生計測で、ローカルモデルが常置指示に行動する場所（system ではなく最初の user メッセージ） |
| [security](docs/ja/security.md) | プロンプトインジェクション対策、秘密情報/PII、公開サービス、破壊的操作の安全設計、ラップできないツール説明文の隔離、sandbox 内のツールチェインキャッシュ、有限領域があるならカーネルへ判定を移し無いなら規則を置かない、dialer 内で閉じる SSRF、サンドボックスが書けるディレクトリには、保証されたディレクトリから開いた os.Root 越しに触る、資格情報がリダイレクトを追ってよいのは要求を開始したドメインの内側だけ、大文字小文字を区別しないディスクで場所を名前でなく実体で比べる —— まだ存在しない場所も含め、名前はディスクと同じやり方で同一視する、ffmpeg はインタプリタ（入力の形式を固め、書いて読み戻すものは呼び出し側やサンドボックスが書ける場所に置かない） |
| [build-and-packaging](docs/ja/build-and-packaging.md) | CGO クロスビルド、.gitignore の罠、CI 不使用の判断理由 |
| [testing](docs/ja/testing.md) | 実データ E2E、モック設計、失敗注入、MCP テストハーネス、クロスプラットフォーム検証、エラー文言の質、診断時の証拠の質、ログの時刻表現、独立検証パスの収束判定、新しい境界は本物を本物の檻で駆動して検証、モデル比較ベンチの設計、inline TUI の行計算は tmux で実測、複数行を占める出力は自分の下を消す、端末の能力プローブ、モデルが読む文面を機械照合、境目に接する区間の切断漏れ、ヘッドレスのテストとビュー描画カウンタ、ゲートが観測している層の名指し、同時1クライアントのフィクスチャと並列スイート、直さない欠陥の固定、クレジット制チャネルの停止、先取り確保する書き手と完了判定、どの問いにも同じ答えを返すフェイク、kitty のグラフィックスプロトコルは PNG のみ、依存の取り消しが応答されるまで枠を返さない（応答するかは実測） |
| [containers-and-infra](docs/ja/containers-and-infra.md) | macOS の Podman、DuckDB bind mount、matplotlib フォント、ログローテート、SSH 死活監視、case-sensitive ボリュームの禁忌、Podman 上の QEMU/TCG による実ゲスト OS、VMのデバイスを製品の使い方に合わせる、一時ポートの割り当てと束縛の競合、podman の name フィルタはアンカー無しの正規表現 |
| [config-and-io](docs/ja/config-and-io.md) | Bubble Tea の Update 内 Send による凍結、canonical 識別子、strict 設定デコード、データ保持期限、保存先変更と reconcile、ボリューム空き容量、OAuth、ターミナル IO、任意の応答で開いたままになる送信窓、goroutine で読んだ端末問い合わせが残す reader と次の入力、時間の予算は一度だけ宣言、検査済みの数値がゼロの Duration になる、手元のパスをテキスト文法へ通し直す、推測した値を事実と別の印で記録し事実だけを比べる |
| [shell-scripting](docs/ja/shell-scripting.md) | BSD/GNU sed 差、zsh 展開の癖、Bash trap スコープ、置換の罠、bash 3.2 の set -u と空配列、`if` 文の後の `$?` |
| [embedded](docs/ja/embedded.md) | M5Stack / ESP32 の知見 |
| [development-process](docs/ja/development-process.md) | rewrite/refactor 判断、コントリビューション triage、ADR 粒度、ドキュメント作法、アーカイブ済みリポの分離、submodule の破損、報告と制御、移植コードの ADR 引用、upstream への提案が望まれているかの見極め、vendored パッチの上流に対する再生、まず自分のパッチを疑う、記録した教訓を検査へ変える、行番号引用を差分で再マップ、独立検証の回ごとの収束、対の照合ではなく参照を解決する |

英語版は [docs/en/](docs/en/)（日本語が原文）。

## License

[MIT](LICENSE)
