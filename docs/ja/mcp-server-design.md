# MCP サーバー設計

Go による MCP（Model Context Protocol）stdio サーバーの実装、および LLM エージェントから
呼ばれるツール一般の設計で確立した知見。各項目は「事象 → なぜ → 適用方法」。

---

## プロトコル・トランスポート

### MCP プロトコルにツール呼び出しのキャンセルは存在しない

**事象:** 「MCP 利用中のチャット中断が効かない」というユーザー報告。in-flight の
ツール呼び出しを中断する手段を調査した。

**なぜ:** MCP プロトコル 2024-11-05 仕様には client → server のキャンセル通知が無い。
一度 dispatch したツール呼び出しは上流の完走まで走る。ad-hoc なプロトコル拡張は
協力的なサーバーにしか効かず、ブロック中の stdio 読み取りも解除できない。

**適用方法:**
- クライアント側で「待つのをやめる」唯一の手段は**子プロセス kill + stdin close で
  Scan を unblock する**（kill-and-respawn パターン）。context 対応ラッパーは
  `goroutine + select(ctx.Done, result)` + cancel 時に Stop() で kill。
- kill したら**当該サーバーのみ**非同期で再 spawn する責務がクライアント側に要る
  （全サーバー再起動は無関係なサーバーを巻き込む）。
- 上流の副作用（外部 HTTP / DB）は kill しても止まらない（server 側で完走し、client が
  結果を捨てるだけ）。プロトコル制約上回避不能なので、ドキュメントで期待値を揃える。
- 将来 protocol-level cancel が入ったら kill 方式は不要になるので、ラッパーは
  差し替えやすい設計にしておく。

### プロキシ/中継はクライアントのリクエストを未応答で放置しない

**事象:** proxy が上流への転送失敗（期限切れ OAuth トークン）をログに出すだけで
クライアントに応答せず、クライアントが 15 秒ハング → 理由不明の
`read response: EOF` になった。

**なぜ:** 他の失敗経路（tool not found 等）は JSON-RPC エラーを返していたのに、
上流転送失敗の経路だけ応答が抜けていた。非対称な失敗経路は必ず漏れを生む。

**適用方法:**
- 不変条件:「ルーティングがエラーを返したリクエストには必ず JSON-RPC エラー応答を返す」。
  中央のディスパッチ点で `msg.IsRequest()`（id あり）なら error response を書く。
  通知（id 無し）は応答不可なのでログのみ。
- `err.Error()` に実用的理由（`access token expired ... run --login again` 等）を載せれば
  クライアントがそのまま surface できる。
- 二重応答に注意: ハンドラは「応答を書く」か「エラーを返す」の一方のみ。中央キャッチは
  未応答で伝播したエラーだけ拾う。
- タイムアウト経路が既にエラー応答に変換しているなら、send/connect 失敗経路も同じ扱いに
  揃える。

### 子プロセスの失敗は「パイプの症状」ではなく終了ステータスを surface する

**事象:** 子プロセス（MCP サーバー）が macOS に SIGKILL されていたのに、エラーは
`initialize: read response: EOF` としか出ず、環境問題がコードのバグに見えて診断に
多大な時間を要した。

**適用方法:**
- 失敗時に `cmd.Wait()` の結果（`signal: killed` / `exit status N`）をエラーに含める。
  SIGKILL なら「quarantine/同期パス配下なら macOS が kill している可能性」等の対処ヒントを
  添えると一発で切り分く。
- **os/exec の落とし穴**: `cmd.Wait()` は `StdoutPipe`/`StderrPipe` の読み取りと競合させては
  いけない。**パイプ読み取りを所有する goroutine から、読み取りループが EOF で終わった後に
  1 回だけ**呼ぶ。別の Wait 監視 goroutine を立てるのはこの競合を踏む。
- 具体形: stderr ドレイン goroutine が Scan 終了後（=プロセス終了）に
  `exitErr = cmd.Wait(); close(exited)`。失敗経路は「閉じたパイプ系エラー」のときだけ
  `exited` を短時間待って exitErr を wrap。プロセス生存中の失敗（上流の JSON-RPC エラー等）
  では待たない。Stop() は kill のみで二重 Wait を防ぐ。

### 埋め込みネイティブライブラリを持つ stdio サーバーは fd レベルで stdout を隔離する

**事象:** プロセス内に静的リンクしたネイティブライブラリ（C/CGO）が、モデルロード中に
進捗バーを **fd 1 に printf** し、JSON-RPC ストリームを破壊した。TTY では `\r` 更新の
一瞬の行なので見えず、pipe（=MCP）でだけ生バイトが残る。stub build のユニットテストでは
再現せず、実エンジン E2E で初めて発覚した。

**教訓の核:** 「コンソールで見えない ≠ stdout に出ていない」。必ず `1>file 2>file` で
ストリームを分離して実測する。

**適用方法（2 層で守る）:**
- **① 発生源を塞ぐ**: 「コールバック未登録なら自前で stdout 描画」する型のライブラリには、
  **常に no-op でもコールバックを登録**しておく。
- **② fd レベル隔離（defense-in-depth）**: 起動時に本物の stdout を dup して private handle
  を作り、fd 1 を stderr に付け替える。transport は private handle にだけ書く。

```go
saved, _ := syscall.Dup(int(os.Stdout.Fd()))
syscall.Dup2(int(os.Stderr.Fd()), int(os.Stdout.Fd()))
mcpOut := os.NewFile(uintptr(saved), "mcp-stdout") // transport writes here
```

- ログは `os.Stderr` へ（付け替え後は `fmt.Println` も stderr に落ちて無害）。
- 子プロセス方式（外部エンジンを子プロセス駆動）ではこの問題は出ない。埋め込み方式特有。
- 検証はダミー stdio クライアントによる実エンジン E2E が必須（testing.md 参照）。
  汚染の兆候はレスポンスストリーム中の空行/非 JSON 行。

## エラー・スキーマ設計

### ツールエラーは {code, message, details} の構造化 JSON で返す

**事象:** エラーを平文文字列で返すと、LLM クライアントは文字列マッチングでしか
分岐できず、プロンプトも実装も不安定になる。

**適用方法:**
- `isError: true` + 単一 text content の中身を構造化 JSON にする:
  `code`（安定 slug: `"path_not_allowed"` 等）/ `message`（人間可読）/
  `details`（マシン可読の文脈: `{"requested": "bash", "supported": ["python"]}` 等）。
- Go 実装: toolerr パッケージに Error 型 + Code 定数群を集約し、`Is(target) bool` を
  Code 比較で実装して `errors.Is` 互換を保つ。sentinel は package-level 変数。
  server 側は `errors.As` → JSON marshal → text content、非構造化エラーは
  `err.Error()` フォールバック。
- 新しい code の追加は互換変更だが、**既存 code の rename は breaking change**。

### クライアントは inputSchema.enum を事前検証する — サーバー側検証は残す

**事象:** enum 違反の呼び出しがサーバーの handler に到達せず、クライアント側で
`invalid_enum_value` として拒否された（Claude Desktop で実測）。

**適用方法:**
- 列挙可能な引数は `inputSchema.enum` で必ず宣言する（往復不要で拒否される良い UX）。
- **同時にサーバー側でも明示的に検証**する — schema を検証しないクライアント・直接
  JSON-RPC を打つテストハーネスに対する defense-in-depth。
- サーバー側 enum 拒否経路のテストはダミー JSON-RPC ハーネスでバイパスして行う
  （実クライアント経由では到達しない）。

### 画像など複数 content block の返却は sentinel 型 + 明示的返却ツールで

**事象:** ツール handler の戻り値を JSON marshal して単一 text block に詰める標準経路では、
画像（PNG plot 等）を返せない。また「実行後にディレクトリを auto-scan して新規画像を
全部返す」設計は、意図しないファイルの混入・heuristic の難しさ・レスポンスサイズの
予測不能で罠が多い。

**適用方法:**
- **RawResult sentinel パターン**: 専用型（`Content []ContentBlock, IsError bool`）を導入し、
  dispatch 側で `if raw, ok := out.(RawResult); ok` なら複数 block をそのまま返す。
  既存ツールは無変更（後方互換）。
- **「何を返すか」は LLM に明示指定させる** `attach_files(workspace_id, paths)` のような
  返却専用ツールを分ける — コントロール可能・監査可能で、サイズ上限強制や path traversal
  防御（`filepath.Clean` + prefix check）も入れやすい。
- ライブラリ関数を monkey-patch して出力を hook する案は副作用が不透明で debug 困難、却下が無難。

## LLM から呼ばれるツールの設計

### 大きなデータをツール引数で渡さない（作業ディレクトリ経由にする）

**事象:** ドキュメント本文を `content` パラメータで直渡しする設計は、大きいファイルで
破綻した。LLM の function call 引数には実用上のサイズ上限がある（数百 KB 程度、
ローカル LLM ではさらに小さい）。これは**ツール設計の primary failure mode**。

**適用方法:**
- `filename` パラメータ + 共有作業ディレクトリ（/work 等）経由のファイル受け渡しを
  正規パターンにする。前段ツールでファイルを配置し、後段ツールは basename で参照する。
- **パラメータ名の曖昧さに注意**: `content` は LLM が「ユーザーの発話文」と「対象
  ドキュメント本文」を取り違える（実例あり: ユーザーのリクエスト文自体を要約対象として
  渡してきた）。`filename` のような型のはっきりした名前を優先し、説明文に
  「NOT the user's request text」等を明示する。
- **XOR 制約は寛容に解釈**: 排他パラメータで空文字列 `""` は「未指定」扱いにする。
  LLM は「使わない」を明示的な空値で表現することが多い。
- **パストラバーサル防御**: filename は basename のみ許可（`/` や `..` を含んだら拒否）。
  LLM は悪意なくとも絶対パスを生成しうる。
- ツールへの LLM ガイダンスは description フィールドのみで完結させる（ホスト側の内蔵
  プロンプトに特定ツールへの参照を入れると結合度が上がる）。description には
  「いつ選ぶか / どのワークフローで呼ぶか / パラメータの意図 / 排他制約」を書く。
- ラップする CLI が進捗を stderr に出す場合は `--quiet` 等で抑制する。

### ツール結果に内部 ID を露出しない

**事象:** ツール結果に `[Stored as object ID: %s]` を含めたところ、LLM がその ID で
redundant なリンクをチャット応答に書き込むようになった（本体は別経路で表示済みなのに）。

**適用方法:**
- ツール結果には「LLM が後続のツール呼び出しで使う情報」だけを含める
  （次のツールで使う ID は残す）。
- ユーザー側に専用 UI で表示済みのコンテンツの ID は結果から除外する。
- 「表示はこちらで処理した」ことは最小限のステータス
  （"SUCCESS: ... displayed to the user. Reply briefly."）で伝える。

## サーバー実装の構成

### 新規 Go MCP サーバーは実績ある骨格を移植する

**事象:** MCP サーバーの transport / JSON-RPC / protocol routing / エラー型は
サービス内容に依存しない generic な部分で、毎回ゼロから書くのは無駄かつ品質が揺れる。

**適用方法:** 4 パッケージ構成の骨格を確立し、新規サーバーは scaffold 段階でこれを移植する:
- `internal/transport/` — stdio: `bufio.Scanner`（大きめ buffer）+ `json.Encoder` +
  mutex serialised write
- `internal/jsonrpc/` — JSON-RPC 2.0 型 + 標準コード定数
- `internal/mcpserver/` — protocol routing + `RegisterTool` API + 構造化エラー出力
- `internal/toolerr/` — `{Code, Message, Details}` + `errors.Is` by code

移植時の変更は import path、sentinel code 定数の差し替え、不要機能（RawResult 等）の削除
程度。ツールハンドラ層と上流クライアントだけを新規実装する。

### 二重状態（in-memory + disk）の mutation は必ず同じ層を経由させる

**事象:** 長寿命の in-memory セッション + disk ファイルの二重状態で、disk だけ書き換える
mutation（rename 等）が in-memory コピーをバイパスした。その後のほぼ毎アクションで走る
save が古い in-memory 値で disk を上書きし、「リネームが効かない/再起動で戻る」ように
見えた。同型の罠を 2 回踏んだ。

**適用方法:**
- per-session 状態を変更する操作は**すべて状態を所有する層（agent 層）を経由**させる。
  UI bindings から永続化パッケージを直接呼ぶのは禁止（レビュー時の赤旗パターン）。
- 状態所有層に同名メソッドを作り、mutex 保護下で in-memory も更新する。bindings は
  thin pass-through にする。
- **ガード判定の罠**: in-memory 値を読んで判定するガード（`if Title != "New Session"` 等）も
  同じ理由で誤動作する。上書き経路（Mode A）と誤判定経路（Mode B）の両方をテストで固定する。
