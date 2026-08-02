# macOS GUI アプリ

SwiftUI / AppKit ネイティブアプリと Wails（Go + WebView）アプリの開発で踏んだ罠と
確立したパターン。各項目は「事象 → なぜ → 適用方法」。署名・notarization は
[release-engineering.md](release-engineering.md) を参照。

---

## SwiftUI / AppKit（メニューバー常駐アプリを中心に）

### メニューバー常駐アプリは SwiftUI 単体では完結しない

**事象:** メニューバー常駐型（`LSUIElement`）はアクティベーションポリシー、パネル、
フォーカス管理で AppKit の知識が必須になる。

**要点:**
- `.accessory` ポリシーではキーボードフォーカスが取れない → パネル表示時に `.regular` に
  切替、非表示時に `.accessory` に戻す。
- `NSPanel` の `isMovableByWindowBackground` は TextEditor 等と競合する → false にして
  タイトルバーのみドラッグ可に。
- SwiftUI の `@FocusState` + `onAppear` は初回 `NSHostingView` 生成時にタイミングが
  間に合わない → `DispatchQueue.main.asyncAfter` で遅延。
- Info.plist は SPM のリソースバンドルに入れられない → プロジェクトルートに置いて
  Makefile でバンドルにコピー。
- **App Nap がバックグラウンド更新タイマーを凍結する**: ウィンドウ非表示の LSUIElement
  アプリは background 判定され、`Timer.scheduledTimer` の周期更新が止まり表示値が固着する
  （起動直後は正しく、長時間稼働で固着するため気づきにくい）。対応は
  `ProcessInfo.processInfo.beginActivity(options: [.userInitiatedAllowingIdleSystemSleep],
  reason:)` の戻り値をアプリ寿命で保持して App Nap を抑止（システムスリープは許可のまま）。
  ユーザーが見る画面は `.onAppear` で明示再取得すると確実。
- **LSUIElement は `Settings` scene / `SettingsLink` をフォーカスできない** → 設定画面は
  通常の `Window` + `openWindow(id:)` + activate で開く。

### NavigationSplitView に bottom safeAreaInset を付けてはいけない

**事象:** `NavigationSplitView` に `.safeAreaInset(edge: .bottom) { footer }` を付けると、
sidebar 内の下部バー（+/- ボタン等）が footer に塗り潰されて完全に見えなくなる。
この状態のままリリースしてしまった。

**なぜ:** bottom safeAreaInset は split view 自身のフレームは縮めるが **sidebar 列は
縮めない**。sidebar はウィンドウ全高で描き続けるため、下部バーが footer と同じ帯に着地する。

**適用方法:**
- footer は VStack の兄弟として置く:
  `VStack(spacing: 0) { NavigationSplitView {...}; Divider(); footer }`。
- SwiftUI で「隠れた/消えた」系の症状が出たら、geometry を実測して当たりを付ける。
  `NSHostingView` を裸の `NSWindow` に載せた**使い捨てハーネス**を `swiftc` で 1 ファイル・
  数分で作れる（本物の Views/Model ソースを直接 swiftc に渡し App だけ差し替えると、
  本物のビューで before/after を実証できる）。実画面確認は
  `screencapture -x -o -R<x,y,w,h>` で対象矩形を切り出す。`NSView.cacheDisplay` は
  effect view 内部を描かないので使えない。
- この類の描画回帰は unit test 不能なので、罠としてプロジェクトの AGENTS.md に残す。

### IME 変換中（marked text）はライブ処理を走らせない

**事象:** デバウンス自動翻訳が、かな漢字変換の途中でデバウンス満了 → 未確定文字列を処理 →
言語判定不能 → OS のダイアログが出て入力が完全にブロックされる連鎖が起きた。

**適用方法:**
- SwiftUI の `TextEditor` は marked text を露出しない。`NSViewRepresentable` で
  `NSTextView` を自前ラップし `hasMarkedText()` を見るしかない。
- `textDidChange` だけでは不十分 — **確定前後で文字列が変わらない場合は通知が飛ばず**、
  確定を検知できない。`setMarkedText` / `unmarkText` / `insertText` を override して
  変換の開始・終了を直接通知させる。
- 変換確定エッジ（composing true→false）でも処理を起動する。
- 判定ルールは純粋関数に切り出してユニットテストする（実機 IME は自動テスト不能）。
- 「入力言語が判定不能なら自動実行を見送る」ガードも有効。手動実行は絶対にゲートしない。

### NSTextView のキャレットは key window でしか描画されない

**事象:** グローバルホットキーでパネルを開くと、macOS がアプリの activate を拒否し
（他アプリが frontmost のまま）、firstResponder なのにカーソルが出ない。

**適用方法（3 経路すべてを塞ぐ）:**
1. `NSPanel` サブクラスで `canBecomeKey` を明示的に true（`canBecomeMain` は false のまま）。
2. show 時に `makeKey()`、さらに 1 runloop 後に key とフォーカスを再主張する
   （`NSApp.activate()` は非同期かつ拒否されうる。`activate(ignoringOtherApps:)` は
   deprecated、macOS 14+ は `activate()`）。
3. `NSWindow.didBecomeKeyNotification` を監視し、key になった瞬間に firstResponder を
   取り直して `updateInsertionPointStateAndRestartTimer(true)` を呼ぶ。

検証は「他アプリが frontmost のまま」キャレットが見えることをスクリーンキャプチャで確認。

### メニューバー用 NSPanel の罠 2 件

**罠 1: `hidesOnDeactivate = true` でトグルが空振りする。** 自動非表示の際
`window.isVisible` が `true` のまま残るため、`isVisible` ベースのトグルが実際は非表示なのに
`orderOut`（空振り）し、「1 回目のクリックが無反応、2 回目で開く」症状になる。
→ `hidesOnDeactivate` を設定せず、クリックアウェイで閉じたい場合は
`applicationDidResignActive` で自前 `orderOut`（これは `isVisible` を正しく false にする）。

**罠 2: `.nonactivatingPanel` にしないと起動直後 ~30 秒開かない。** 通常の NSPanel は
アプリがアクティブでないと描画されないが、macOS 14+ の focus-stealing 防止で起動直後は
アクティブ化が最大 ~30 秒拒否される。`isVisible == true` で位置も正しいのに画面に出ない。
→ styleMask に `.nonactivatingPanel` を追加（非アクティブでも描画・キーボード入力可）。
NSPopover 系がこの問題を踏まないのは popover がアクティブ化不要で表示されるため。

付随の知見:
- `setFrameAutosaveName` でサイズを記憶するパネルは、表示時にサイズを現在スクリーンの
  `visibleFrame` にクランプする（大画面で保存したサイズが小画面で画面外化する）。
- この種の診断はファイル書き出しログ + **実クリック**が確実。AppleScript の
  `click menu bar item` は NSStatusItem の button action を発火しない。

### 連続アニメするメニューバーアイコンは NSStatusItem で

**事象:** SwiftUI `MenuBarExtra` の label は静止画像としてレンダリングされ state 変化時のみ
更新される設計で、連続アニメーションには向かない。

**適用方法:** `NSStatusItem` + layer-backed `NSView` に CAShapeLayer を載せ、
`lineDashPhase` 等のレイヤープロパティをアニメする（実測 ~0.3% CPU）。宣言的なパネルや
チャートは SwiftUI を `NSHostingView` で NSPopover/NSWindow に載せるハイブリッド構成。
UI 型は `@MainActor`、Timer は target/selector 版を使うと Swift 6 の capture エラーを
避けられる。アプリ寿命の常駐 view は deinit での timer 破棄不要。

### NSPopover 内の SwiftUI パネルは開いた時だけ生成する

**事象:** `TimelineView(.animation)` を含むパネルを eager 生成して保持すると、popover が
**非表示でも**レイアウトエンジンが毎フレーム再計算し続け、アイドルで高 CPU（実測 ~12%）。

**適用方法:** show 分岐で `NSHostingController(rootView:)` をその場で生成し、
`popoverDidClose` で `contentViewController = nil`。パネル内の装飾アニメは避け、静的
ゲージ（`Shape.trim` 等）にする — 動きはメニューバー側（CAShapeLayer）が担当する。
診断は `sample <pid>` で SwiftUICore の LayoutEngine が支配的かを見る。

### SwiftUI の List/OutlineGroup でツリーの D&D を作らない

**事象:** 挿入位置インジケータ付きのツリー D&D を SwiftUI で実装して撤回した。
(1) `.dropDestination` はドロップ確定時にしかポインタ位置を返さず挿入線が出せない。
(2) `DropDelegate` は位置は取れるが**セッション終了の通知が信用できない** —
`dropExited` / `performDrop` が来る保証がなく、挿入線が消えずに残り、最終的に
「更新が途絶えたら消す」ウォッチドッグに退避することになった。

**適用方法:**
- 挿入インジケータ付き並べ替えが要件なら**最初から `NSOutlineView`（AppKit）**で書く。
- その投資に見合わなければ**機能ごと落とす**判断をする。
- 併発する罠: ビューのボディからファイルシステムを走査しない（SwiftUI はボディを高頻度で
  再評価するため、ドラッグ中に事実上フリーズする）。ツリーはキャッシュし revision で更新。
- 一般化: 「Finder でできること」を再実装する前に本当に必要か問う。

### GUI アプリは版数を画面に出す

**事象:** メニューバー常駐アプリは CLI の `--version` に相当する導線が無く、入れ忘れると
ユーザーが版数を知る手段がゼロになる。不具合報告で「どのビルドか」が分からないのは実害。

**適用方法:**
- scaffold 時点でバージョン表示を入れる。値は
  `Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString")`
  （ビルド時に `git describe` から注入、バンドル外実行は `"dev"` フォールバック）。
- **verbatim で出す** — `-dirty` / `-N-g<sha>` 接尾辞も削らない。報告者が正確なビルドを
  貼れるほうが有用。`textSelection(.enabled)` でコピー可能に。
- 表示場所: メニュー型は About 項目、パネル型はパネル内に直接。
- フォールバック判定は純関数に切り出してテストする。

---

## Wails（Go + WebView）

### window.alert() は確実に表示されない

**事象:** Wails v2（macOS WKWebView）では `window.alert()` / `confirm()` / `prompt()` が
無反応（サイレント）になる。エラー通知経路が全滅していたが、エラー経路は稀にしか
踏まれないため長期間気づかなかった。

**適用方法:** ユーザ向け通知は フロント共通ヘルパ → Go binding →
`wailsRuntime.MessageDialog`（Info/Error）で表示する。確認系はインライン UI
（2 クリック削除、N 秒 Confirm）。結果通知を Go 側で出す設計にするとフロントは値の取得 +
リフレッシュだけで済む。GUI のエラー表示は「エラーを意図的に起こして実機で出ること」を
確認するまで信用しない。

### OnStartup でブロッキング処理を同期実行しない

**事象:** `OnStartup` 内で子プロセス spawn + ハンドシェイクやコンテナエンジン probe を
同期実行した結果、起動が遅延・ハング。さらに **Wails は OnStartup 実行中も frontend の
binding 呼び出しを並行ディスパッチする**ため、frontend 初期化が未構築のバックエンド
state を呼んでレース失敗した。

**適用方法（非ブロッキング startup の型）:**
1. 窓サイズは生成時に確定（`wails.Run` の `options.App{Width,Height}`）。
2. コンストラクタは即返しし、外部依存 init は `StartBackground(ctx)` に分離して goroutine で。
3. 2 段 readiness シグナル（`app:ready` / `tools:ready`）を EventsEmit + イベント
   取りこぼし対策に query 用 binding も用意する。
4. 入力 UI は `tools:ready` までゲート（半初期化状態への送信レースを根絶）。
5. probe には必ずタイムアウト（`context.WithTimeout`）。
6. goroutine 化で共有になったフィールドは mutex + `go test -race`。

### Menu と mac.About を明示設定しないと標準メニュー/About が消える

**事象:** `Menu:` 未設定だと最小限のデフォルトメニューのみで About 項目が無く、
`Cmd+C/V/Z` も native shortcut として動かないケースが出る。長期間ユーザーが GUI から
バージョン確認できない状態で出荷していた。

**適用方法:** scaffold 時に `menu.AppMenu()` + `menu.EditMenu()` + `menu.WindowMenu()` を
組んだ `Menu:` と、`Mac.About`（`*mac.AboutInfo` にタイトル / バージョン文字列 /
アイコン）をセットで仕込む。メニューハンドラから runtime API を呼ぶ場合、menu 構築時には
ctx がまだ無いので、startup で捕捉した ctx を closure 経由で参照する。

### appicon.png を差し替えても Windows の icon.ico は再生成されない

**事象:** macOS の .icns は毎ビルド `appicon.png` から再生成されるが、
`build/windows/icon.ico` は `wails init` 時に一度生成されたきりの静的ファイル。
カスタムアイコン導入後も Windows .exe だけデフォルト「W」ロゴのまま出荷していた。

**適用方法:** アイコン差し替え時は `icon.ico` も再生成する。Pillow で生成できる:

```python
from PIL import Image
src = Image.open('build/appicon.png').convert('RGBA')
src.save('build/windows/icon.ico', format='ICO',
         sizes=[(256,256),(128,128),(64,64),(48,48),(32,32),(24,24),(16,16)])
```

### 初回から multi-platform の release ターゲットを Makefile に揃える

**事象:** `wails build` を呼ぶだけの単一ターゲットだと、リリース時に手動でクロスビルド +
rename する運用になり、「Intel 版だけバンドル名が違う」といった手作業起因の不整合を産んだ。

**適用方法:** scaffold 段階で `build-darwin-arm64` / `build-darwin-amd64` /
`build-windows-amd64` / `build-all` / `package: build-all` を用意する。各 per-arch ビルドは
先頭で `rm -rf build/bin` して stale .app の誤 package を防ぐ。package は notarize + staple
の後、stage ディレクトリで canonical 名に rename してから arch-suffix zip 化。
windows/amd64 は Apple Silicon ホストから Wails v2.12 で問題なくクロスビルドできる。

### 半透明ウィンドウは避ける

**事象:** `WebviewIsTransparent: true` + CSS rgba 背景で「native 風」の見た目にしたところ、
長文表示でデスクトップが透けて読みづらく、真の blur は private API が必要と判明。

**適用方法:** `WebviewIsTransparent: false` でスタートする。surface-level の CSS token は
`rgb()`、レイヤー系（opaque 親の上に乗るもの）は rgba のままでよい。
`TitlebarAppearsTransparent: true` は macOS native の挙動範囲内なので OK。

### go-duckdb は no_duckdb_arrow タグが必要

**事象:** go-duckdb を Wails に組み込むと Arrow の CGO リンクエラー
（`Undefined symbols: ArrowArrayIsReleased` 等）が出る。

**適用方法:** `database/sql` インターフェースのみ使うなら
`wails build -tags no_duckdb_arrow` で Arrow を除外する。Makefile に記載しておく。

---

## 共通パターン

### GUI アプリに CLI サブコマンドを同居させる（単一バイナリ）

**事象:** GUI アプリにバックグラウンド CLI モードを追加する際、別バイナリにすると
ビルドターゲット増加、.app への同梱、パス解決が複雑化する。

**適用方法:** `main()` で `os.Args[1]` を確認し、`wails.Run()` の前にルーティングする。
CLI モードでは `wails.Run()` が呼ばれないため GUI オーバーヘッドはゼロ。自身の spawn は
`os.Executable()` でそのまま解決できる。
