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

### メニューバーの NSPopover は表示直後に makeKey() する

**事象:** ステータスアイテムのクリックは accessory (LSUIElement) アプリを**アクティブ化
しない**ため、`popover.show(relativeTo:)` しただけの popover ウィンドウは key にならず、
macOS はその素材を非アクティブ状態で描画する。macOS 26 の Liquid Glass では、これが
「暗く沈んだ半透明シート」としてはっきり見える（実測: パネル本体の平均輝度 192 → 224 / 255）。
非アクティブ描画は macOS 26 で急に目立つようになった側面があり、それ以前から同じコードが
動いていても「ある日から暗くなった」と観測される。

**適用方法:**
- `show(relativeTo:)` の直後に `popover.contentViewController?.view.window?.makeKey()`。
  これだけで `NSApp.activate` + `makeKey()` と**ピクセル単位で同一**の描画になる
  （`makeKey()` 自体がアプリをアクティブ化するため、別途 activate を呼ぶ必要はない）。
- 副作用として `.transient` の外側クリック判定が壊れる（アクティブ化したアプリでは元々
  信用できない）。**明示的な global + local マウスダウンモニタで自前クローズする前提**で
  導入すること。
- 検証は実機で: 合成クリックで開く → `screencapture -x -o -R <bounds>`（背景込みの
  合成結果）で撮り、修正前後の平均輝度を比較する。**ウィンドウ単体撮影
  (`screencapture -l <windowid>`) は背景が抜けるため、半透明素材の見え方の判定には
  使えない**（実測で「暗い」偽陽性が出る）。

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

### UNUserNotificationCenterDelegate が無いと、前面にいる間はバナーが出ない

**事象:** 通知は**通知センターには積まれる**のに、バナーが一切表示されない。OS の通知設定で
スタイルを「一時的」から「持続的」に変えると表示される。おやすみモード等は無関係。

**なぜ:** `UNUserNotificationCenterDelegate` の `willPresent` に応答しないと、macOS は
**アプリがフォアグラウンドにある間の通知を黙って通知センターに入れるだけ**にする。
メニューバー常駐アプリは「前面ではない」と思いがちだが、設定ウィンドウなど自前の
ウィンドウにフォーカスがあれば前面であり、**「テスト送信」ボタンを押した瞬間はまさに前面**。
そのため一番確認したい場面で一番出ない。

**適用方法:**
- delegate を実装し `willPresent` で `[.banner, .list]` を返す。`.list` も返すことで、
  離席中に見逃した通知が通知センターに残る。
- **配信より先に install する**。`UNUserNotificationCenter.delegate` は weak 参照なので、
  アプリ寿命の間**強参照で保持**する（ローカル変数に入れると即解放され、症状が再発する）。
- 症状からの近道: 「通知センターには入るがバナーが出ない」= ほぼ確実に delegate 未設定。
  権限やおやすみモードを疑う前にここを見る。

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
- **そもそも延命が必要かを先に疑う**。次の項目のとおり、通知の投げ方を変えると
  この延命は丸ごと不要になり、上記の事故クラス自体が到達不能になった。

### 通知のためにプロセスを延命しない — trigger 付きでスケジュールする

**事象:** ワンショット起動のアプリが完了バナーを出した直後に終了すると、
**バナーが即座に消える**。そのため「バナーが出ている間だけ生かしておく」実装
（`.accessory` に降格して数秒後に `terminate`）に流れる。それが前項の事故の温床。

**なぜ:** `UNUserNotificationCenter` に `trigger: nil` で投げた通知は
`willPresent` 経由で**そのプロセスが提示するもの**として扱われ、プロセスの終了と
運命を共にする。ところが `UNTimeIntervalNotificationTrigger` を付けて
**スケジュール**すると、提示の所有者が `notificationd` に移る。実測（署名済み
実バイナリ、時刻指定スクリーンショット）:

| 方式 | プロセス生存 | バナー |
|---|---|---|
| 即時 post、提示直後に terminate | 0.16→1.17 s | **t=1.5 s で消滅** |
| trigger(0.5 s)、`add` 完了で即 terminate | 0.16→**0.57 s** | t=1.5 s / t=5.0 s とも表示 |
| trigger(0.1 s)、同上 | 0.12→**0.47 s** | t=2.0 s 表示 |

つまり制約は「通知」ではなく「即時提示」の性質だった。延命は不要である以上に
有害で、前項の被害はすべてその数秒の窓に起きていた。

**適用方法:**
- 完了通知は `UNTimeIntervalNotificationTrigger(timeInterval: 0.1,
  repeats: false)` でスケジュールし、**`add` の完了ハンドラが返った時点で
  `terminate` してよい**。提示を待つ必要はない。0.1 秒の遅れは体感できない。
- 待つべきなのは提示ではなく **`add` の成功**。XPC 往復の前に死ぬと通知自体が
  登録されない。認可（`requestAuthorization`）もキャッシュせずこの連鎖の中で解決する。
- `willPresent` デリゲートは**常駐時の前面表示のために残す**（返さないと自アプリが
  前面のときバナーが出ない）。
- 検証は実バイナリ + `screencapture` の時刻指定ショットで。通知センターに項目が
  残っているかではなく、**画面にバナーが出ているか**を見ないと区別できない。
- 副作用: アプリ終了後にバナーをクリックすると**アプリが再起動する**。
  `didReceive` で結果を Finder に表示するなど、クリックに意味を持たせる。
- 一般化: **回避策を堅牢化する前に、回避対象の制約を実測で確かめる**。
  API の別の形が制約ごと消してくれることがある。

### 通知クリックは Bundle ID で解決される — 常駐アプリは単一インスタンスを強制する

**事象:** 常駐メニューバーアプリの通知バナーをクリックしたら、実行中の
インスタンスが前面に来るのではなく**アプリが 2 つ起動した**（メニューバーに
2 個並び、ポーリングも二重化）。（status-lens、2026-08）

**なぜ:** バナーのクリックで notificationd は「この Bundle ID のアプリを開け」と
LaunchServices に依頼する。開発マシンでは同一 Bundle ID の .app が複数箇所に
登録されがち — ビルド出力ディレクトリの開発ビルド、リリース検証で展開した
コピー、インストール版。LaunchServices は登録済みコピーのどれかを解決するので、
**実行中のコピーと別の場所が選ばれると新プロセスとして起動する**（Xcode の
DerivedData ビルドが起動してしまう現象と同型）。ビルドのたびに出力ディレクトリの
.app が再登録されるため、lsregister での掃除は恒久策にならない。

**適用方法:**
- 常駐 GUI は**二層ガード**を入れる:
  1. Info.plist に `LSMultipleInstancesProhibited: true` — LaunchServices 経由の
     起動（通知クリック・`open`）を LS レベルで抑止
  2. 起動時に `NSRunningApplication.runningApplications(withBundleIdentifier:)`
     で自分以外のインスタンスを検出したら stderr に一行出して exit 0 —
     直接バイナリ実行や `open -n` をカバー。判定は「pid 列 → 判定」の純粋関数に
     切ればユニットテストできる
- Bundle ID が nil（素の開発バイナリ）のときはガードを素通しにする —
  そもそも列挙ができない
- SwiftUI の `@main struct X: App` には Scene 構築前にコードを走らせる場所が
  ない: stored property の初期化子（`@StateObject` のモデル。多くは更新ループ
  起動込み）は `init()` 本体より先に走る。`@main` を小さな
  `enum Main { static func main() }` に移してガードを先に実行し、その後
  `X.main()` を呼ぶ — 重複インスタンスはモデルが動き出す前に終了する
  （2026-08 に GUI 10 本へ展開）
- **CLI を同居させたバイナリは GUI 起動経路だけをガードする。** チェックは
  引数ディスパッチの後（GUI 分岐の中）に置き、`main` の先頭には置かない:
  CLI サブコマンドまでガードされると、GUI 稼働中は「another instance」を出して
  exit 0 する — 実作業を頼まれたのに黙って何もしない状態で、定期実行ジョブで
  最悪（nvme-lens で検出。main 先頭ガードだと `sample` が空振りする。
  zip-porter の pack/unpack を並行可能なままにしたのも同じ理由）
- ガードは**起動される側**に効く。未ガードの旧版がインストールされたまま残って
  いる間は、逆向き（開発ビルド実行中に旧インストール版が起動）だけ守れない —
  修正版の配布までがこの修正の一部
- 検証は実バンドルで 2 経路: 直接実行（ガードのメッセージと exit 0）と `open`
  （LS 経路で二重起動しないこと）。どちらも実行中のインスタンスを殺さずに行える
- 副作用: 開発ビルドの動作確認は、先にインストール版を quit してから

### 複数選択の一括処理は「1 リクエスト = 1 報告」にする

**事象:** Finder で N 個のファイルを選択して開くと、アプリは 1 件ずつ処理して
1 件ずつ完了バナーを出していた。**macOS は同一アプリの新しいバナーで前のバナーを
置き換える**ため、3 件処理しても画面上で読めるのは最後の 1 枚だけ。残りは通知
センターに N 件溜まる。加えて Finder 表示 N 回、ダイアログ設定なら OK が N 回、
保存先「毎回確認」なら選択パネルが N 回、パスワードが N 回。

**なぜ:** Finder の複数選択は **1 回の `application(_:open:)` に全 URL が
まとめて**届く（実測）。「1 ファイル = 1 ジョブ」という内部構造をそのまま報告
単位にしてしまうと、ユーザーが 1 回の意思表示で頼んだことが N 回の中断になる。

**適用方法:**
- 1 リクエスト（1 回の open イベント / 1 回のドロップ）= **1 進捗バー・1 質問・
  1 完了報告・1 Finder 表示**。進捗は各項目のサイズで加重して全体の分母を作り、
  「3 件中 2 件目 — foo.zip」を添える。
- **1 件の失敗で全体を止めない**。残りを処理し、最後に「3 個中 2 個」と失敗一覧を
  出す。失敗を含む結果はバナーではなくダイアログにする（バナーは失敗を伝える場所
  ではない）。
- 確認・入力の使い回し: 保存先は 1 回、パスワードは次の項目にまず試してから聞く。
- 報告の集約ロジックは**純粋な値型**に切り出す（何をバナーで済ませ、何をダイアログで
  止めるかの規則こそテストする価値がある）。
- 一覧を打ち切るなら**打ち切ったことを必ず書く**（「…ほか N 件」）。黙って切ると
  完全な一覧に見える。

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

### メニューバーのアイコンは `button.image` に入れる — attributed string の中では template が効かない

**事象:** メニューバーのシンボルが、正常時だけ他のシステム項目より淡いグレーに見えた。
`isTemplate = true` を設定していたので原因が分からず、明暗の壁紙を切り替えても変わらない。
実際には**画像を `NSAttributedString` の `NSTextAttachment` として埋め込んでいた**。

**なぜ:** `isTemplate` は `NSStatusItem.button.image` に対してのみ尊重される。
attributed string に埋め込んだ画像は指定された色のまま描画され、テンプレート扱いされない。
そのため「メニューバーの色に従う」つもりの指定が無視され、アプリ文脈の `labelColor` が
そのまま焼き込まれていた。メニューバーの実際の色と近いが一致せず、**間違いに見えるほど近く、
グレーに読めるほど違う**状態になる。

**適用方法:**

- シンボルは `button.image` に置き、テキストは `button.title` / `attributedTitle` に置いて
  `imagePosition` で並べる。テキスト添付に画像を混ぜない
- 色そのものが伝えたい状態（警告のオレンジ、危険の赤）では `isTemplate = false` にする。
  テンプレートはメニューバーに塗り直されるため、その色が消える
- **正常時に緑を点けない。** 99% の時間点いている色は、目がアイコンを読み飛ばすよう
  学習させる。そして異常時の一度を見逃す。正常はメニューバー自身の色に任せ、
  色が付くのは注意が要るときだけにする

### SF Symbol 名は実在を検証してから使う — 存在しない名前は無言で退化する

**事象:** `internaldrive.fill.badge.exclamationmark` をアイコン名に使った。もっともらしい
名前だが**実在しない**。`NSImage(systemSymbolName:)` が nil を返し、フォールバックの
"•" が描かれる——メニューバーが黙って無意味になる。

**なぜ:** SF Symbols の命名は規則的に見えて、実際にはシンボルごとに用意されている
バリエーションが違う。`externaldrive.fill.badge.exclamationmark` は存在するが
`internaldrive` 版は無い、といったことが起きる。コンパイルは通り、実行時エラーも出ず、
例外も投げられない。

**適用方法:**

- レンダラーが出しうる**全シンボル名を配列で公開し、それが解決することをテストで固定する**。
  `NSImage(systemSymbolName:accessibilityDescription:) != nil` を assert するだけでよい
- そのために executable ターゲットにもテストターゲットを作る。ロジックを Core に寄せていても、
  シンボル名の選択はビュー側に残る
- フォールバックは無害な見た目にせず、**壊れていると分かる形**にするか、そもそも到達しない
  ことをテストで保証する

### ビューの寸法を「今日の中身」で測って固定しない

**事象:** メニューバーアプリのパネル・チャート内部・設定ウィンドウで、作った時点の中身に
合わせた固定値（`frame(height: 560)` 等）を書いた。その後セクションやグラフを追加するたびに
**3 回とも表示が壊れた**（中身が切れる、要素が潰れる、はみ出す）。

**なぜ:** 固定値は「書いた瞬間の中身」に対する正解でしかない。中身は増える。しかも
壊れ方が「切れる」「潰れる」なので、レイアウトの不具合ではなく別の問題に見える。
さらに同じ寸法を 2 箇所に書くと（ビューの高さと、それを配置する側の高さ）、片方だけ直して
気づかないまま食い違う。

**適用方法:**

- **下限と理想値を宣言し、固定しない**。`frame(minHeight:idealHeight:)` +
  ウィンドウは `.resizable`。中身が増えても壊れず、利用者が広げられる
- 一致していなければならない寸法は**定数を 1 つ定義して両方から参照する**
- 内部でテキスト行を確保するビューは、その行が使われないケース（外側が既に表示している等）で
  確保をやめる。使わない領域を確保し続けると、本体の描画領域が痩せる
- 残してよい固定値は「入れ物が自分で決めるべき寸法」（ポップオーバーの幅など）だけ

### Apple Mail の複数メッセージ D&D は旧式 file promise プロトコルでしか受け取れない

**事象:** `NSFilePromiseReceiver.readableDraggedTypes` を登録したドロップターゲットが、
Apple Mail の 1 通ドラッグは受理するのに、複数通ドラッグは型不一致でドラッグごと
拒否した。

**なぜ:** ドラッグ用ペーストボード（`NSPasteboard(name: .drag)`）の実測で、Mail の
1 通ドラッグはモダン promise 型と pre-10.12 の旧型
（`com.apple.pasteboard.promised-file-url` / `NSPromiseContentsPboardType`）の両方を
載せるが、**複数通ドラッグは旧型のみ**を載せ、モダン型はペーストボードから消える。
さらに `receivePromisedFiles` の reader ブロックは Mail 相手では発火しないことが
多い（既知のプラットフォーム不具合）。

**適用方法:**
- `registerForDraggedTypes` に `com.apple.pasteboard.promised-file-url` を追加し、
  `performDragOperation` 内で非推奨の `namesOfPromisedFilesDropped(atDestination:)`
  で解決する。deprecated だが Mail の複数通 promise を受けられる唯一の API で、
  **正確な約束ファイル名の数**も得られる（1 通ドロップの完了判定も速くなる）。
- ファイル実体は非同期に書き込まれる。完了判定は「約束数が揃い、全ファイルの
  サイズが一定時間変化しない」で行い、ハードデッドラインで**必ず**結果を出す
  （0 件なら失敗として UI に通知する。無言のタイムアウトにしない）。
- ドロップごとに一意な一時サブディレクトリを掘ると、同名衝突・連続ドロップの
  競合・ディレクトリ差分スナップショットが全て不要になる。
- 診断には drag ペーストボードを 100ms 間隔でポーリングして型一覧と
  `canReadObject(forClasses: [NSFilePromiseReceiver.self])` の結果を出力する
  小さな sniffer スクリプトが決定的に効く。ドロップは不要で、ドラッグ開始
  → Esc キャンセルだけで観測できる。

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

### エラー UX を設計する前に、フレームワークのエラー型の全ケース表を実測する

**症状:** ある翻訳アプリは、どの失敗に対しても
`"Couldn't translate — the language model may still be downloading. "` ＋
`error.localizedDescription` の 1 行しか出していなかった。未対応の言語ペアも、
サービス内部エラーも、本当にモデルが無い場合も、まったく同じ表示になり、
しかも「ダウンロード中かもしれない」という前半は大半のケースで**誤り**だった。

**なぜ:** `localizedDescription` はフレームワーク自身のシステムダイアログ向けに
書かれており、こちらの UI 向けではない。macOS 26.5 SDK で実測すると
`TranslationError` は **8 ケース中 7 ケースで `"Unable to Translate"`** を返し
（異なるのは `nothingToTranslate` のみ）、全ケースが `NSError` domain
`Translation.TranslationError` の **code 1** にブリッジされる。区別できる文言は
`failureReason` 側にあり、誰もそれを読んでいなかった。
「`localizedDescription` には情報がある」という思い込みそのものがバグ。

**適用方法:**
- エラー表示を設計する前に、**全ケース**について
  `localizedDescription` / `failureReason` / `NSError.domain` / `.code` を印字する
  使い捨てプログラムを書く。ケース一覧はまず SDK のインターフェースから取る:
  `$(xcrun --show-sdk-path)/System/Library/Frameworks/X.framework/Modules/*.swiftmodule/*.swiftinterface`。
- 独自 `~=` を持つ `struct` のエラー型は、enum のように形で `switch` できない。
  **N×N のマッチ行列**で本当に判別するかを確認する（完全な対角行列であること、
  無関係なエラーがどれにもマッチしないこと）。`TranslationError` は合格した。
- マッチは `classify(Error) -> 自前 enum` 1 箇所に閉じ込め（フレームワークの
  エラー型に触れてよい唯一の場所）、文言生成はユニットテスト可能な**純関数**にする。
- 文言は 3 段で書く: 原因 / **どの UI 要素を操作すれば直るかを名指しした**対処
  （上流の文言はこちらの UI を知らないので書けない）/ 不具合報告用の選択可能な
  技術タグ。
- **上流の文言を捨てない。** 未知のエラーは
  `failureReason ?? localizedDescription` ＋ domain/code を必ず載せる。捨てると、
  **まさに想定できなかったケース**で元のバグが再発する。
- **推測した原因を固定の前置きにしない。** 「〜かもしれません」の固定文言は、
  そうでない全ケースで積極的に誤誘導する。

### 「意図的に何もしない」状態も、UI で名乗れなければならない

**症状:** ある翻訳アプリが「動作しているのかどうか分からない」と報告された。
このアプリは翻訳を見送る**正しい**判断を 5 つ積み上げていた — IME 変換中、
言語判定に足りない短さ、600 ms のデバウンス、source == target のエコー、
初回のモデルダウンロード — そして**そのすべてが無言**だった。
正しい待機はハングと見分けがつかない。

**なぜ:** アプリには既に `isTranslating: Bool` があり、どのビューも読んでいなかった。
読ませても解決しない。Bool は「なぜ何も起きていないのか」を言えず、
問われていたのはまさにその「なぜ」だった。

**適用方法:**
- 状態は `isLoading: Bool` ではなく `enum Phase` でモデル化する。Bool が要るなら
  phase からの**導出**にして（`isWorking: phase == .preparing || .translating`）、
  ずれる二つ目の真実を作らない。
- phase → 表示（記号 / 文言 / スピナー / トーン）の写像は**純関数**にして
  ユニットテストする。「全ケースが空でない文言を返す」テストを 1 本置いておくと、
  将来 phase を足したときに無言の状態が生まれない。
- **静かに処理を飛ばす早期 `return` の直前では、必ず phase を立てる。**
  レビュー観点はこれ — 各早期 return の上に状態代入があるか。
- 状態だけでなく**出口**を書く:「まだ言語を判定できません — もう少し入力するか、
  左でピン留めしてください」。
- ステータス行は**無条件に描画**してレイアウトが跳ねないようにし、スピナーと
  アイコンは同一の枠に排他配置してテキストが横にずれないようにする。
- **待機にスピナーを出さない。** スピナーは進行中の主張。held / pending には
  静的アイコンを使う。

これはパネルにバージョン文字列を出すのと同じ原則である。メニューも About 項目も
ログファイルも無いメニューバーアプリでは、**画面に出ているものが情報経路の全て**
— 状態についても、失敗の技術的原因についても。
