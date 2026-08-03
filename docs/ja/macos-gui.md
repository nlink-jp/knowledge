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

### `.app` の中で SwiftPM の `Bundle.module` を使わない

**事象:** ビルドしたマシンでは完璧に動くのに、それ以外のマシンでは起動直後に必ず落ちる。
`_assertionFailure` での `EXC_BREAKPOINT`、スタックに `NSBundle.module`。notarize・staple
済みリリースを新規インストールしても同じ。テストでも、ローカル実行でも、`spctl` でも
検出できない。

**理由:** `resources: [.process("Resources")]` を持つターゲットに SwiftPM が生成する
アクセサが試す経路は 2 つだけ:

1. `Bundle.main.bundleURL/<Package>_<Target>.bundle` — 素の CLI 実行ファイルなら正しいが、
   `.app` では誤り。バンドルの実際のインストール先はルート直下ではなく
   `Contents/Resources`。
2. **コンパイル時に焼き込まれた `.build/…/release/…` の絶対パス。**

経路 1 はアプリバンドルでは決して当たらないので、解決に成功しているのは常に経路 2 —
つまりビルドしたマシンにしか存在しないパス。ローカル開発からは構造的に見えない不具合で、
ローカルで検証すればするほど確信が深まるという最悪の性質を持つ。

**適用方法:**
- アプリターゲットで `Bundle.module` を参照しない。`Bundle.main.resourceURL`（`.app` の
  レイアウト）→ `Bundle.main.bundleURL`（素の実行ファイル）→ コードバンドルのある
  ディレクトリ（`Bundle(for:).bundleURL.deletingLastPathComponent()`、`swift test` を
  カバー）の順に探すロケータを自前で書く。
- 最後のフォールバックは trap ではなく `Bundle.main` にする。ローカライズテーブルが
  無ければ英語のソースキーに退化させればよい。翻訳を失うことはアプリを殺す理由にならない。
- 探索順とヒット判定は純関数に切り出してユニットテストする。実行時の呼び出しは
  テスト済みロジックの薄いラッパーになる。
- **リリース検証手順に組み込む:** `.build/*/release/*.bundle` を退避してから
  `dist/<App>.app/Contents/MacOS/<App>` を直接起動する。この一手順で他マシンを再現できる。
  リリース用アーカイブをクリーンなディレクトリに展開し、そのバイナリを起動すれば更に確実。
- 組織横断の棚卸しは `grep -l 'resources:' Package.swift`。リソースを宣言している
  Swift ターゲットはすべて容疑者。

*出典: 出荷済みの macOS アプリ 2 本が、作者以外の全ユーザーの初回インストールで起動不能
だった。いずれも全テスト・notarize・ビルド機での手動 QA を通過していた。*

---

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

### MenuBarExtra はコンテンツに「高さを押し付ける」

**事象:** `MenuBarExtra(style: .window)` のパネルが、ヘッダーとフッターだけが縦に密着した
状態で表示され、中身が丸ごと消える。あるいは行同士が重なり、チャートの塗りが隣の行を覆う。

**なぜ:** このパネルは「中身に合わせて伸びる」のではなく、**コンテンツの ideal サイズで
自分の大きさを決め、その高さをコンテンツに渡す**。`ScrollView` は無限に伸縮できるので
**ideal 高さが 0**、つまり高さ 0 に潰れる。`.frame(maxHeight:)` は効かない（max は
「要求された高さ」の上限を与えるだけで、誰も要求していない）。必要より小さい高さを
渡された `VStack` は子を圧縮するため、行が重なる。

**適用方法:**
- ルートに `.fixedSize(horizontal: false, vertical: true)` を付けて「必要な高さを主張」させる。
- パネル内に `ScrollView` を置くなら**具体的な高さ**を与える。行数から見積もる純関数にし、
  潰れない下限と画面外に出ない上限を持たせてユニットテストで守る。見積もりは厳密でなくてよい。
- 可能なら**そもそも件数に上限を設けて `ScrollView` を使わない**。問題が構造的に消える。
- 同じ罠は greedy な view 全般（`Spacer` / `GeometryReader` / `List`）に当てはまる。
- グラデーションの area fill には `.clipped()` も。潰れても隣を侵さない方がマシ。

### List 内 ForEach の id は Section 内ではなく List 全体で一意にする

**事象:** グループ分けしたリストで、ある項目のチェックを入れると**同じ種類の項目が全部
チェックされたように見える**。

**なぜ:** `ForEach(names, id: \.self)` のように「Section ごとに同じ値が現れうるもの」を
id にすると、SwiftUI が別々の行を同一の行として扱う。保存されているデータは正しく、
壊れているのはビューの identity だけなので、モデルのテストでは検出できない。

**適用方法:**
- 行の id は「その行を一意にする複合キー」にする。`Identifiable` な値型を作り
  （例: `id = "\(groupID)/\(itemKey)"`）、`ForEach(items)` で回す。
- 行の配列を**純関数**として切り出す（`static func rows(for:) -> [Item]`）と、
  id の一意性をユニットテストで守れる。SwiftUI 本体はテストできなくても材料はテストできる。
- 症状からの近道: 「1つ操作すると同じ種類の項目が全部反応する」= ほぼ確実に id 重複。
  データ側を疑う前にビューの identity を見る。

### 定期更新の Timer は .common run loop モードに登録する

**事象:** パネルやメニューを**開いている間だけ**表示が更新されなくなる。見るために開いた
瞬間に固まる、という最悪の挙動。

**なぜ:** `Timer.scheduledTimer` は `.default` run loop モードにしか登録されない。メニューや
ポップオーバーの追跡中、run loop はそのモードを離れるため、タイマーが発火しない。

**適用方法:** `let t = Timer(timeInterval:repeats:){...}` を作り
`RunLoop.main.add(t, forMode: .common)` で登録する。

### 経過時間ラベルは TimelineView で駆動する

**事象:** 「最終更新: 0s ago」のような鮮度表示が、**永久に「0s ago」のまま**になる。
止まっていないことを示すために付けたラベルが、止まった時計そのものになる。

**なぜ:** SwiftUI は観測している値が変わった時しか再描画しない。レンダリング時に
`Date()` を読む書き方だと、元データが更新される瞬間に「0s」と描いてそこで止まり、
次の更新まで再評価されない。

**適用方法:** `TimelineView(.periodic(from: .now, by: 1))` でテキスト自身を秒ごとに
再評価する。他の部分は巻き添えで再描画されない。

### 「いいえ」と「分からない」を区別できない status でコントロールを disable しない

**事象:** ログイン項目（`SMAppService`）のスイッチが**誰も ON にできない**。

**なぜ:** `SMAppService.mainApp.status` は**一度も登録されていないアプリに `.notFound` を
返す**（`.notRegistered` ではない）。これを「この複製は登録できない」と解釈してスイッチを
無効化すると、**初回は必ず disable** になり、登録する手段が存在しなくなる。初回こそが
唯一必要な操作なのに、それを封じてしまう。

**適用方法:**
- 迷ったら **disable せず、実行させて結果を報告する**。「やってみないと分からない」なら
  やらせるのが正しい。
- 曖昧な状態は「不可能」ではなく**「まだ有効になっていない」**に畳む。
- 実行後に**状態が変わったかを検証**し、変わっていなければそう言う。「エラーは出ないが
  スイッチが戻る」が最悪。
- **操作した画面にエラーを出す**。別画面にしか出ないと「押しても何も起きない」に見える。
- 外部の status 列挙をそのまま UI に持ち込まず、**UI が取るべき行動**に写像する。

### OS の許可要求は「使う瞬間」ではなく「ユーザーが意思表示した瞬間」に出す

**事象:** 通知トグルを ON にしているのに通知が来ず、**OS の通知設定一覧にアプリが現れない**。

**なぜ:** `requestAuthorization` をアラート発生時まで遅延させると、条件が一度も成立しない
うちは要求されない。OS にアプリの記録が無いので設定一覧にも出ない。結果として「設定
できない／動作を事前確認できない／条件が来なければ永久に無音」という、確認しようのない
約束をするトグルになる。

**適用方法:**
- トグルを ON にした時点で要求する。ユーザーが意思表示した瞬間であり、プロンプトが文脈上
  意味を持つ唯一の瞬間で、そこでアプリが OS に登録される。
- **既に ON なのに未要求という過去状態を起動時に検知して救う**（版数を跨いだ移行）。
- 拒否されている場合は UI がそう述べ、解除できる設定ペインを開く導線を出す。
  ON なのに何も届かないスイッチを黙って表示しない。
- 通知に限らず位置情報・カレンダー等、OS 許可全般に当てはまる。

### 別の ObservableObject の変更は、観測していないビューには届かない

**事象:** 設定画面のチェックボックスは即座に変わるのに、**メニューバーの表示だけが次の
定期更新まで古いまま**。

**なぜ:** 表示に使う値（モデル）と、表示のしかたを決める選択（設定オブジェクト）が別々の
`ObservableObject` だと、後者の変更は前者しか観測していないビューを再描画しない。

**適用方法:** そのビューが実際に依存している**両方**を `@ObservedObject` で持つ。
環境に注入していない場所（`MenuBarExtra` の label 等）は特に漏れやすい。

### 「見た目だけ終了して裏で生きている」プロセスは、必ず復帰可能にする

**事象:** Finder から起動されたワンショット処理の完了後、通知バナーを消さないために
アプリを数秒だけ延命していた（Dock からは消す = `.accessory`）。その数秒の間に
**次のファイルをダブルクリックされると、前のジョブのタイマーが新しいジョブを殺す**。
実測で確認した被害:
- 700MB の展開が書き込み中に kill され、**543MB の切り詰められたファイルが
  エラー表示なしで残る**（成功と見分けがつかない）
- 暗号化 ZIP のパスワード入力プロンプトが、**入力開始から約 2.8 秒で消滅**
- Dock アイコンをクリックしてアプリを残そうとしても、約 2 秒後にウィンドウごと消える

**なぜ:** 遅延 `terminate` は「まだ見えていない未来」についての決定なのに、
`DispatchQueue.main.asyncAfter { NSApp.terminate(nil) }` は
**ハンドルも再判定も持たない**。プロセスは Dock から消えていても
**open イベントを受け取り続ける**ため、LaunchServices は死刑執行待ちのプロセスに
平然と新しい仕事を配送する。さらに `.accessory` への降格を戻す経路が無いと、
新しい仕事を受けたのに「終了したふり」のままになる。

**適用方法:**
- 遅延終了は必ず **(a) キャンセル可能なハンドル**（`DispatchWorkItem`）として持ち、
  プロセスに新しい存在理由を与える**全経路でキャンセル**する（open イベント、
  reopen＝Dock クリック、設定を開く、ユーザー操作）。
- **(b) 発火時にもう一度判定する**。スケジュール時の判断は数秒前の世界のもの。
  判定を**純粋関数**に切り出すと、2 回の評価が同じ規則を使うことが保証でき、
  テストも書ける（zip-porter の `OneShotQuit.decide`）。
- **「処理中」は他のあらゆる条件に優先**させる。前のジョブのバナー残り時間は、
  今走っているジョブについて何も語らない。
- 「処理中に来た要求」は**ビープで捨てずにキューに積む**。ウィンドウを出さない
  Finder 起動経路では、ビープは聞こえず何も表示されないので**黙って消える**。
- 逆方向の確認も必須: 修正後に**最後のジョブの後でちゃんと終了するか**を必ず試す。
  早すぎる kill を直してプロセスリークを作っては本末転倒。

### 標準の編集ショートカットはメインメニューに項目が無いと存在しない

**事象:** パスワード入力欄で **⌘V が効かない**。⌘W でウィンドウも閉じられない。
コード上に該当処理が無いので、ブレークポイントを置く先すら無い。

**なぜ:** macOS は ⌘X/⌘C/⌘V/⌘A/⌘Z や ⌘W を、**メインメニューの key equivalent を
経由して**ファーストレスポンダに配送する。テキストフィールド自身がキーを解釈するわけ
ではない。メニューを自前で `NSMenu` から組む（xib を使わない）アプリで Edit メニューを
省くと、`paste:` を持つ項目が存在せず、キーストロークはどこにも届かない。App メニューと
Window メニューだけ作って「動いている」ように見えるのが罠。

**適用方法:**
- 自前でメニューを組んだら **Edit（Undo/Redo/Cut/Copy/Paste/Delete/Select All）と
  File > Close を必ず入れる**。入力欄が 1 つでもあるなら Edit は必須。
- **メニューバーが描くのはトップレベル `NSMenuItem` 自身の title であり、submenu の
  title ではない**。`NSMenuItem()` に submenu を挿しただけでは、中身が完成していても
  **メニューごと不可視**になる。App メニュー（プロセス名）と Window メニュー
  （`NSApp.windowsMenu`）は AppKit が特別扱いするため無題でも出てしまい、これが
  「無題でよい」という誤解を生む。
- メニュー構築関数から `NSApp` へのアクセスを追い出す（`build()` と
  `install(into:)` に分ける）。**XCTest 環境では `NSApp` は nil** で、触ると
  クラッシュする。分離すればメニューをテストで検査でき、key equivalent の結線を
  自動テストで固定できる。
- 検証は実機のメニューバーで。System Events の
  `value of attribute "AXMenuItemCmdChar"` で結線を、セキュアフィールドへの
  ⌘V 後の `length of (value of field)` で実際の貼り付けを確認できる。

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
