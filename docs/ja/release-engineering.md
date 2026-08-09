# リリースエンジニアリング

macOS の署名・notarization、リリースアーカイブ、Homebrew tap 配布に関する知見集。
各項目は「事象 → なぜ → 適用方法」の構造で、実プロジェクトで踏んだ事例から抽出している。
規範的なルールは [CONVENTIONS.md](https://github.com/nlink-jp/.github/blob/main/CONVENTIONS.md)
（§Release Archive Standard, §Code Signing and Notarization）を参照。本書はその背後にある
「なぜ」と、ルール化しきれない落とし穴を扱う。

---

## tar.gz 配布でも darwin バイナリは notarize できる（temp-zip パターン）

**事象:** Apple の notary service（`xcrun notarytool submit`）は zip / dmg / pkg
コンテナしか受け付けず、tar.gz は直接送れない。

**なぜ有効か:** notary ticket はバイナリの **CDHash（中身のハッシュ）をキーに Apple の
サーバに保存される**。最終配布フォーマットが何であっても、実行時に macOS が CDHash で
問い合わせて ticket を見つければ notarized として扱われる。

**適用方法:** packaging の前に「temp で zip 化 → submit → zip 破棄」を挟む。

```make
dist-darwin: cross-build-darwin
	@for arch in arm64 amd64; do \
		cp dist/$(BINARY)-darwin-$$arch /tmp/$(BINARY) && \
		(cd /tmp && zip -j /tmp/$(BINARY)-notary.zip $(BINARY)) && \
		scripts/notarize-darwin.sh /tmp/$(BINARY)-notary.zip "$(NOTARY_PROFILE)"; \
		rc=$$?; \
		rm -f /tmp/$(BINARY) /tmp/$(BINARY)-notary.zip; \
		test $$rc -eq 0; \
	done
	# この後は無修正の tar.gz / .deb / .rpm packaging でよい — バイナリは既に notarized
```

- `rm -f` で notarize 失敗時も temp を掃除し、`test $$rc -eq 0` で失敗を伝播する。
- ticket は CDHash 紐付けなので、submit 後にファイル名を変えても zip を作り直しても有効。

## notarytool 403 "required agreement missing/expired" は契約失効であってキー失効ではない

**事象:** それまで通っていた notarization が突然一律失敗し、ラッパースクリプトは
「Keychain profile not found」と報告した。

**なぜ:** スクリプトが事前 probe（`notarytool history`）の**終了コードだけ**を見て失敗理由を
決め打ちしていた。直接 `xcrun notarytool history --keychain-profile <profile>` を叩くと実際は
`403 A required agreement is missing or has expired` — Apple Developer Program のライセンス契約が
更新され、Account Holder が未同意だっただけで、キーもプロファイルも有効だった。

**適用方法:**
- ラッパーの失敗表示を鵜呑みにせず、必ず `notarytool history` を直接実行して実エラーを見る。
- `403 ... agreement ...` なら developer.apple.com/account で更新契約に同意する
  （App Store Connect の Agreements も確認）。数分で反映され、キー再登録は不要。
- ラッパースクリプトは probe の stderr を捕捉して**実エラーを必ず表示**し、
  403/agreement を検出したら契約再同意を案内するよう修正した（親切な誤診断は誤診断より悪い）。
- `notarytool history` の事前 probe は flaky（成功直後に別 arch で偽 fail することがある）。
  スクリプト経由で落ちたら `notarytool submit --wait` を直接叩いて迂回する。
- notarytool の keychain profile は **login キーチェーン＝マシンローカル**。
  別マシンには同期されないので、ビルドマシンごとに `store-credentials` が必要。

## クラウド同期フォルダ配下のバイナリは SIGKILL され得る（provenance xattr）

**事象:** Dropbox 同期フォルダ配下の `~/bin` に Go バイナリを `cp` すると、起動即
`zsh: killed`（SIGKILL）。同じバイナリが `dist/` 直下（同期外）では動く。

**なぜ:** Dropbox の FileProvider extension は、sync フォルダにファイルが landing した時点で
`com.apple.provenance` 等の xattr を付与する（**同一マシン内のローカル `cp` でも付く**）。
`com.apple.provenance` は system-protected で `xattr -d` でも削除不可。弱い署名
（ad-hoc / linker-signed）と provenance の組合せを Gatekeeper が SIGKILL する。
この挙動は macOS のマイナーアップデートで強化され、「以前は動いていた構成」が突然壊れた。

**適用方法:**
- 同期フォルダ配下に置いた後に `codesign --force --sign - <binary>` で再署名すると通る
  （provenance は残るが、署名が「このマシンが今 sign した」ものになる）。
  同期先の**各マシンで**再署名が必要（署名は sync では再付与されない）。
- `cp` は宛先に新規 inode を作るので xattr が新規付与されるが、Finder の同一ボリューム内
  「移動」は rename（inode 保持）なので発火しない。`ditto` も可。素の `cp` を避ける。
- `spctl --add` は最新 macOS で deprecated（"This operation is no longer supported"）。
- **これは環境/配置の問題でコードバグではない**。子プロセスの即死
  （`read response: EOF` / `broken pipe`）を見たら、まず `<binary> --version` の
  exit code（137 = SIGKILL）と配置場所（同期フォルダ配下か）を確認する。
- プロジェクトの Makefile に個人環境向けの `install:` target（`~/bin` 決め打ち cp +
  codesign）を入れない。プロジェクトは `make build` → `dist/` まで。個人環境への配置は
  dotfiles 側のスクリプトで吸収する。

## GUI .app の署名は CLI と別パイプライン

**事象:** CLI 用の署名/notarize スクリプトは単体 Mach-O 前提で、.app バンドルには流用できない。

**要点**（正規手順は CONVENTIONS.md §Code Signing → GUI app signing）:
- **WebView 系（Wails / Tauri）は entitlements 最小 2 つが必須**:
  `com.apple.security.cs.allow-jit` + `com.apple.security.cs.allow-unsigned-executable-memory`。
  無いと WKWebView の JS が Hardened Runtime に殺され、フロントエンドが**無言で真っ白**になる。
- **ネイティブ Swift / AppKit は JIT entitlements 不要**。Hardened Runtime のみで動く。
  localhost への HTTP も Info.plist の `NSAppTransportSecurity` 側で対応でき、entitlement 不要。
- **`cp -r` は禁止、`ditto` を使う**: バンドル署名は xattr に格納されるため、`cp -r` は署名を
  壊し「Code Signature Invalid」で起動即死する。配布 zip 化も `ditto -c -k --keepParent`。
- **署名順序:** build → deep 署名（`--force --deep --options runtime --timestamp
  --entitlements`）→ ditto → notarize（.app は temp zip 化して submit）→
  `stapler staple`。バンドル形式は staple できるのでオフライン first-launch でもダイアログが
  出ない。CLI 単体 Mach-O は staple 不可（オンライン check が走る）。
- **Tauri は `tauri build` が env var（`APPLE_SIGNING_IDENTITY`）を見て bundler 内で署名**する。
  Tauri 内蔵 notarize は独自の env（Apple ID + パスワード）を要求し notarytool の keychain
  profile を使えないため、無効のままにして自前で xcrun notarytool を回すのが楽。
  DMG ファイル名は Rust triplet arch（`aarch64`）で `uname -m`（`arm64`）と一致しない。
  `package` target を `build` 依存にすると再ビルドで staple が剥がれる。
- Swift Package Manager ベースの GUI は `.app` バンドル構造（`Contents/MacOS/` 等）を
  Makefile で手動組み立てし、最後に codesign する。

## リリースアーカイブは「公開物を実ダウンロード」して検証する

**事象:** ビルドログとローカル dist が正しくても、公開資産の中身が意図通りとは限らない。
全 org 監査（52 リポ / 171 アーカイブ）で、ある zip の README/LICENSE 欠落を検出した。

**適用方法:**
- `gh release download` で**公開資産を実 DL → 展開して検査**する。検査項目:
  canonical バイナリ名 / `file` での arch 一致 / README.md + LICENSE 同梱 / darwin の署名。
- **`.app` の zip は `ditto -x -k` で展開する。`unzip` は禁止** — unzip は symlink/xattr を
  壊し、`codesign --verify --deep` が「a sealed resource is missing or invalid」と
  **誤失敗**する（元 zip は健全なのに）。
- **`spctl -a -t exec` は bare CLI Mach-O を仕様上 reject する**（"the code is valid but does
  not seem to be an app"）— notarize の有無と無関係。CLI の確認は
  `codesign --verify --strict` + `Authority=Developer ID Application` + Identifier 一致で行う。
  `spctl` で "accepted, source=Notarized Developer ID" と評価できるのは `.app` だけ。
- **`git tag`（lightweight）は `git describe`（annotated 限定）から見えない**。Makefile の
  版数導出は `git describe --tags`（lightweight 込み）を使う。

## リリース zip 内の実行ファイルは canonical 名にする

**事象:** `zip -j` は渡したファイルの basename をそのままエントリ名にするため、per-arch
成果物（`tool-darwin-arm64`）を直接固めると zip 内のバイナリに arch サフィックスが残り、
利用者が配備時に rename を強いられる。32 ツールで一括修正になった。

**適用方法:** stage-and-zip パターン — per-arch 成果物を canonical 名でステージコピー
してから固める。

```make
$(eval STAGE=dist/_pkg-$(GOOS)-$(GOARCH)) \
rm -rf $(STAGE) && mkdir -p $(STAGE) ; \
cp dist/$(BINARY)-$(GOOS)-$(GOARCH)$(EXT) $(STAGE)/$(BINARY)$(EXT) ; \
zip -j $(ARCHIVE) $(STAGE)/$(BINARY)$(EXT) LICENSE README.md ; \
rm -rf $(STAGE) ; \
```

- 単一 Mach-O の `cp` は署名を保持する（.app バンドルと違い ditto 不要）。
- notarization は CDHash 紐付けでファイル名に依存しないため、rename しても通る。

## CLI は `--version` フラグに応答必須（brew test が叩く）

**事象:** `version` サブコマンドしか持たないツールは、共有 formula テンプレートの
`assert_match version.to_s, shell_output("#{bin}/<name> --version")` により `brew test` が
"unknown flag" で失敗する。`brew install` は成功するので tap 掲載後に初めて露見し、
4 本まとめてパッチ版を切ることになった。

**適用方法:**
- scaffold 時点で `--version` を通す（cobra なら `rootCmd.Version = Version`）。
  サブコマンドは互換のため残し、`SetVersionTemplate` で出力を一致させる。
- 回帰テストでは **`rootCmd.Version` をテスト内で代入しない** — 代入するとフラグが生えて
  壊れたバイナリでも通ってしまう。本番 init での結線（`rootCmd.Version != ""`）を検査する。

## アップロード前にビルド済みバイナリの `--version` を確認する

**事象:** `VERSION ?= dev` のハードコードのまま `make build-all` を実行し、全リリース
バイナリのバージョンが `dev` になった状態でアップロードしてしまった。

**適用方法:**
- Makefile は `VERSION ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo dev)`
  を最初から使う（`?=` なので手動指定も可能なまま）。
- `make build-all` 直後に `dist/<binary>-darwin-arm64 --version` を実行し、期待する
  バージョン文字列と一致してからアップロードする。リリースチェックリストに含める。

## 一度公開したバージョン番号を再利用しない（fix-forward）

**事象:** GitHub Release が存在するタグを削除して貼り直すと、リリースが Draft に戻り
"Latest" 指定も失われ、手動復旧が必要になった。

**なぜ:** 利用者が既にダウンロードしたバイナリと同じバージョン番号の別バイナリが世に出ると、
バージョン番号が識別子として機能しなくなる。

**適用方法:**
- 既公開資産に影響する修正でも再リリースせず、次のパッチ/マイナーリリースで自然に直す
  （fix-forward）。
- タグを消す前に必ず `gh release list` でリリースの有無を確認する。リリースが無いタグに
  限り貼り直してよい。

## prebuilt-binary Homebrew tap の作法と検証

**事象:** ソースビルドすると Developer ID 署名が剥がれる。tap は署名+notarize 済み
リリース zip をそのまま入れる prebuilt 方式にする。

**適用方法:**
- **formula（Go CLI）**: `depends_on arch: :arm64` + `depends_on :macos`、
  `bin.install "<name>"`。資産名にバージョンが中間に入る形式では brew の自動 version 検出が
  効かないため `url` ハードコード + `version` 明示。`brew style` の ComponentsOrder は
  `url` が `version` より前を要求するので、`url` に `#{version}` 補間は使えない。
- **cask（GUI .app）**: `depends_on macos: :big_sur`（文字列形は style が symbol 形に矯正）、
  `zap trash: [...]`。
- **cask が notarization の真テスト**: formula は brew が quarantine xattr を剥がすが、
  cask は保持する → `spctl -a -t exec` がフル Gatekeeper 判定を行う。
  クリーンマシンで "accepted, source=Notarized Developer ID" になれば署名保持の決定的証明。
- **クリーン VM が無いときの代替検証**: 公開 Release から zip を DL → 自分で
  `com.apple.quarantine` xattr を付与 → `codesign --verify --strict` + quarantine 付きで実行。
  quarantined な notarized バイナリが実行できれば Gatekeeper のオンライン判定を通過した証拠。
- **公開前検証は local tap で GitHub 不要**: tap リポの working copy を
  `$(brew --repository)/Library/Taps/<org>/homebrew-tap` に clone すれば
  `brew install <org>/tap/<name>` が動く（url は公開 release 資産を指すので DL 可能）。
  `brew audit --tap` は Casks/ ディレクトリが無いと cask を検査しない。
- ssh 越しの brew は PATH に無い（`/opt/homebrew/bin` は `.zprofile` 経由）→
  `eval "$(/opt/homebrew/bin/brew shellenv)"` を先頭に。cask install は
  `--appdir="$HOME/Applications"` で sudo プロンプト（非対話 ssh でハング）を回避できるが、
  **`--appdir` は uninstall には渡せない**（receipt から解決される）。
- 後片付けは uninstall → untap の順（installed 成果物が残っていると untap が拒否される）。

## dist/ のバイナリ再署名は、そこから起動中の常駐プロセスを殺す

**事象:** `make build` が `dist/<binary>` を上書き + `codesign --force` で再署名した結果、
そのバイナリから起動していた常駐 daemon が macOS に kill され、データ記録が 30 分以上
無言で欠落した。

**なぜ:** 実行中バイナリのコード署名が変わると macOS がプロセスを落とす。LaunchAgent 未登録
だと `KeepAlive` による自動復帰も効かない。

**適用方法:**
- 常駐 daemon は `dist/` からではなく **`.app` 同梱パス**（`Contents/Resources/<binary>`）から
  起動させる。`make build` の影響を受けず、LaunchAgent の `KeepAlive` で復帰する。
- ビルドを伴う作業の前後で daemon の status（最終サンプル時刻等）を確認する。
  長い欠落は自分のビルドが落とした可能性を疑う。

## 同梱物のあるパッケージは「サイズ」で真っ先に嘘がばれる

**事象:** GUI アプリに CLI バイナリを同梱するリリースで、`make clean` 直後に
`make package` を実行したところ、**同梱が黙ってスキップされたまま notarize まで通過**した。
zip が 1.2MB（正常なら約 10MB）だったことで、アップロード前の検証で気づいた。

**なぜ:** 同梱処理が `[ -x "$CLI_BIN" ]` のような存在チェックで、無ければ警告を出して
続行する作りだった。`make clean` は素の名前のバイナリを消し、`package` が依存する
`build-all` はプラットフォーム接尾辞付きの名前しか作らないため、参照先が存在しなかった。
署名も notarize も「中身が欠けていること」は検出しない。

**適用方法:**
- **アップロード前に必ずアーカイブを展開し、中身を数える**。同梱物の有無、実行ファイルの
  `--version`、署名（`spctl --assess`）を実物に対して確認する。
- 同梱物のサイズは最も安価な健全性指標。桁が変わったら疑う。
- 依存ステップが「無ければ警告して続行」なら、**リリース経路では失敗させる**方が安全。
  開発時の利便性のための緩さが、そのままリリース事故になる。

## 同梱するバイナリの版数文字列は明示的に固定する

**事象:** タグ後に Makefile の1行だけ直したところ、`git describe` が
`v0.1.0-1-g<sha>` を返し、**同梱バイナリの版数表示がリリース版でない文字列**になった。

**なぜ:** 版数を `git describe --tags` から取る構成では、タグ以降に何かコミットすると
（それがコンパイル対象でなくても）版数文字列が変わる。

**適用方法:** 同梱物をビルドするときは `make build VERSION=vX.Y.Z` のように**明示指定**する。
コンパイル対象に差分が無いことを `git diff --stat <tag>..HEAD` で確認したうえで、
リリース版の文字列を名乗らせる。

## scaffold 時に書いた状態記述は、リリースが更新してくれない

**事象:** 「プレリリース」「未リリース」と書かれた README を持つツールが 3 本見つかった。
いずれも実際にはリリース済みで、うち 1 本は **4 回リリースし、Homebrew tap にも載り、
その `brew install` コマンドを README 自身が 2 行下に載せていた**。もう 1 本は
「未リリース。公開後は以下で入ります」と書いた直後に、既に動く brew コマンドを
置いていた。

**なぜ:** リリースチェックリストが見るのは CHANGELOG・バージョン・署名・アーカイブで、
**scaffold 時に書いた散文は誰も見ない**。しかも「まだ出ていない」は書いた瞬間は正しく、
最初のリリースで初めて誤りになる — 書いた本人がその瞬間に立ち会わない種類の腐り方をする。
同種のものとして、README の機能説明が**後で実測して否定された挙動**を宣伝し続けていた例も
あった（別ツールの org profile 行）。

**適用方法:**

- **状態を散文で書かない。** リリース済みかどうかはインストール手順の有無で伝わる。
  版数も書かない（`git tag` が唯一の情報源）
- どうしても書くなら、**リリース手順が触るファイルにだけ書く**（CHANGELOG 等）。
  README の冒頭バナーは誰の手順にも含まれない
- **機械的に検出できる。** 「リリースが 1 件以上あるのに README に
  `未リリース|プレリリース|not yet released` が含まれる」は grep + `gh release list` で
  判定でき、組織横断チェックに足せる
