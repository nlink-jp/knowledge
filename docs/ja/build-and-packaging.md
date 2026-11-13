# ビルド・パッケージング

Makefile、クロスコンパイル、.gitignore、CI 方針に関する知見。各項目は
「事象 → なぜ → 適用方法」。基本ルール（`make build` → `dist/`、版数は `git describe`）は
CONVENTIONS.md §Build Conventions を参照。

---

## CGO + 上流 prebuilt static library の Windows クロスコンパイルは UCRT + ranlib

**事象:** go-duckdb（prebuilt `libduckdb.a` 同梱）を Debian ベースのコンテナから Windows に
クロスコンパイルすると、デフォルトの `gcc-mingw-w64-x86-64` では多段階の linker error が
連鎖する: `cannot find -lstdc++` → `undefined reference to pthread_*` →
C++ ABI 不一致（`_Mbstatet` の name mangling）→ `archive has no index`。

**なぜ:**
- **真の原因は prebuilt library が UCRT（Universal C Runtime）ABI でコンパイルされている
  こと**。Windows 10 以降は UCRT が標準で、デフォルトの mingw（MSVCRT 系）では C++ の
  name mangling が食い違う。
- さらに Debian の `gcc-mingw-w64-ucrt64` パッケージは archive index を欠いた `.a` を含む
  （packaging の問題）ため、`ranlib` で全 archive に index を付け直す必要がある。

**適用方法:** 症状を 1 段ずつ追わず、最初から UCRT toolchain + ranlib-all で組む:

```makefile
build-windows:
	$(CONTAINER) run --rm -v "$(CURDIR):/workspace:z" -w /workspace $(GO_IMAGE) \
		bash -c 'apt-get update -qq && apt-get install -y -q gcc-mingw-w64-ucrt64 g++-mingw-w64-ucrt64 \
			&& find /usr/lib/gcc/x86_64-w64-mingw32ucrt /usr/x86_64-w64-mingw32ucrt -name "*.a" -exec x86_64-w64-mingw32ucrt-ranlib {} + \
			&& GOOS=windows GOARCH=amd64 CGO_ENABLED=1 \
			CC=x86_64-w64-mingw32ucrt-gcc CXX=x86_64-w64-mingw32ucrt-g++ \
			go build -ldflags "-X main.version=$(VERSION)" -o $(DIST_DIR)/$(BINARY)-windows-amd64.exe .'
```

一般原理: **Windows 向け prebuilt C++ static library を使うときは、その library が
どの C Runtime（UCRT / MSVCRT）で作られているかを最初に確認する**（upstream の build
手順や CI workflow に書いてあることが多い）。

## .gitignore にバイナリ名を書かない — cmd/<name>/ ごと消える

**事象:** `.gitignore` にバイナリ名をそのまま書いた結果、`cmd/<name>/` ディレクトリも
除外されて main.go が git 未追跡のままになり、コードロストが複数プロジェクトで発生した。

**適用方法:**
- ビルド出力は `dist/` に統一し、`.gitignore` には `dist/` のみ記載する。バイナリ名の
  パターンは一切書かない。
- ただし標準的な不要ファイルの除外は入れる: macOS junk（`.DS_Store` `._*` 等）+
  エディタ残骸（`*.swp` `*~`）+ `.claude/`。「バイナリ名を書かない」と「標準 junk は書く」
  は両立する。

## リリースをローカルビルドにする判断（GitHub Actions 不使用）

**事象:** リリースの CI/CD 化（GitHub Actions）を検討・提案される機会は繰り返しあるが、
意図的にローカルマシンビルドを維持している。

**なぜ（判断理由として記録）:**
1. **macOS runner のコスト**: Actions の課金で macOS は Linux の 10 倍。個人 OSS の
   リリース頻度でも無料枠を簡単に超える。
2. **署名鍵の保管場所**: Developer ID 証明書と notary 用 API キーはローカル Keychain
   のみに置きたい。GitHub Secrets に置くと漏洩時の blast radius が広く、鍵を CI runner に
   渡す経路自体が攻撃面になる。Apple の規約上も signing identity は keyholder の
   sole control が想定されている。

**適用方法:**
- `make package` 一発で cross-compile + codesign + notarize + zip まで完結させることで、
  手作業ミスのリスクを下げる（CI の代替はスクリプト化）。
- 善意の CI 化 PR（release workflow 追加等）には、動機を認めつつこの方針を説明して
  丁寧に close する。「lint/test だけの CI」も例外を作ると判断がブレるため、
  ローカルの `make check` で完結する方針を貫く。

---

## vendored 依存はローリングタグではなく不変のバージョンタグに固定する

**事象:** sherpa-onnx を submodule 化して静的ビルドしようとしたところ、cmake が
onnxruntime のアーカイブを取得する段で失敗した。ダウンロードした zip 自体は健全
（`unzip -t` 通過）なのに、上流が pin している SHA256 と一致しない。submodule の HEAD は
master 追従で、`xcframework` というタグの周辺を指していた。

**なぜ:** `xcframework` は上流の**ローリングタグ**で、同一タグに対してアセットが随時
再アップロードされる（GitHub のリリースページでアセットの日付が混在する）。コミットも
「onnxruntime のバージョンを 1.27.1 に上げている最中」の状態だった。**可変な参照点に
依存していた**のが原因で、上流の不具合ではない。`v1.13.4` という通常のバージョンタグでは
ハッシュが一致した。

**適用方法:** submodule は **`vX.Y.Z` 形式の不変タグ**に固定する。ローリングタグ
（`latest`、`nightly`、`xcframework` のような機能名タグ）と master の任意コミットは、
第三者が中身を差し替えられる点で再現性がない。

上流が prebuilt アーカイブを configure 時に取りに行く構成では、**URL とハッシュを上流の
定義から抽出して自前で取得し、ハッシュを検証してから配置する**のも有効
（cmake 内蔵のダウンローダが環境によって失敗する問題も同時に回避できる）:

```makefile
# 括弧を含む正規表現を $(shell ...) に書くと make のパーサが壊れるので避ける
ORT_URL  = $(shell grep -m1 'set.onnxruntime_URL' $(ORT_CMAKE) | cut -d'"' -f2)
ORT_HASH = $(shell grep -m1 'set.onnxruntime_HASH' $(ORT_CMAKE) | cut -d'"' -f2 | cut -d= -f2)
$(ORT_ZIP):
	@curl -fsSL --retry 3 -o $@ "$(ORT_URL)"
	@echo "$(ORT_HASH)  $@" | shasum -a 256 -c - || (rm -f $@; exit 1)
```

---

## 「バージョンを PIN する」は改竄対策にならないことがある — 露出を先に棚卸しする

**事象:** 「サプライチェーン攻撃が怖いので submodule をリリースタグに PIN したい」という
方針で作業を始めたが、実際に棚卸ししたところ**PIN しようとしていた場所は問題ではなく、
別のところに穴があった**。

**なぜ:**
- **git submodule は既に commit SHA で固定されており、SHA は content-addressed**。
  タグ名より改竄に強い（タグは動かせる）。リリースタグにして増えるのは主に**監査可能性**
  （CVE との突き合わせ）であって、改竄耐性そのものではない
- さらに、リリースタグへ「戻す」と**既知の脆弱性修正を失う**ことがあった。実際、
  最新リリースから先の未リリースコミットに、ヒープ外読み出しとスタックバッファ
  オーバーフローの修正が 2 件含まれていた（いずれもこのツールが untrusted input を
  食わせる経路）
- 本当の穴は**実行時にダウンロードするモデルファイルの検証がサイズのみ**だったこと。
  サイズは差し替え可能な攻撃者が容易に維持できるうえ、そのファイルは C++ パーサに渡る

**適用方法:** 方針から入らず、**取得物を全部並べて「いつ・何で検証しているか」の表を作る**。

| 取得物 | いつ | 検証 |
|---|---|---|
| submodule | ビルド時 | git SHA（content-addressed） |
| prebuilt アーカイブ | ビルド時 | SHA256 |
| **モデル/データ** | **実行時** | **← ここが抜けやすい** |

そのうえで:
- **不変性は SHA で担保、リリースタグは監査可能性のため**と目的を分けて考える。
  脆弱性修正が未リリースなら、修正入りのコミットに留まるほうが安全
- ダウンロード物は**ハッシュを pin して検証**する。多くの配布元は公開している
  （Hugging Face なら API の `lfs.sha256`）
- **「既存ファイルがあるので DL を省く」高速パスを必ず塞ぐ**。ここをサイズだけで
  素通りさせると、検証したつもりの経路が一番危ない
- 検証は**リネームの前**に行う。失敗したファイルが一瞬でも正規のパスに存在しないようにする
- テストは**同一サイズで中身だけ差し替えた**ケースを明示的に落とす

**副産物:** ハッシュを集める過程で、カタログの 2 エントリが**バイト単位で同一**だと判明した
（第三者ミラーが作者版と同じ中身を別バージョン名で配っていた）。しかもそれが既定だった。
**既定で入るものは上流の作者から取る。** ハッシュ収集は棚卸しとしても効く。

## リポジトリ内のモジュールキャッシュは、再帰的な書き換えに巻き込まれる — `go mod verify` で見つけ、`go clean -modcache` で作り直す

**症状:** 文書しか変えていないのに `make test` が `no required module provides package github.com/spf13/cobra` で失敗した。
`go.mod` には書いてあり、キャッシュにも実体がある（2026-09-22、scat）。

**原因:** Makefile が `GOMODCACHE` をリポジトリ内の `.cache/go-mod` に向けていた（サンドボックス対策）。ワークスペースに
再帰的な整形がかかった過去の事故で、その中の依存ソースまで書き換えられていた。`go mod verify` が
`dir has been modified` と名指しした（`cobra`・`x/sys`）。`.cache/` は git 管理外なので、差分にも出ない。

**どう適用するか:**
- 説明のつかないモジュール解決の失敗は、まず `GOMODCACHE=<その場所> go mod verify`。
- 作り直しは `GOMODCACHE=<その場所> go clean -modcache`。キャッシュは読み取り専用で作られるので `rm -rf`
  （`make clean` の中身がそれでも）は `Directory not empty` で途中止まりになる。
- キャッシュをリポジトリ内に置くなら、整形・一括置換の対象から確実に外れているかを確かめる。


### macOSで作るLinux配布tarにmacOSのメタデータを混ぜない — `COPYFILE_DISABLE=1` と `--no-xattrs` の両方が要る

**事象:** Go CLIの4環境向けrelease（2026-09-22）で、macOS上の `tar -czf` が
`LICENSE`、`README.md`、binaryに加えて `._LICENSE`、`._README.md`、`._<binary>` を収録した。
ビルド・署名・公証は成功していたが、archiveの収録名を検査して初めて見つかった。

**なぜ:** macOSのtarは拡張属性を2つの形でarchiveへ持ち込み、1つの設定は片方しか止めない。
`COPYFILE_DISABLE=1` はAppleDoubleの補助エントリ（`._name`）を抑えるが、bsdtarは属性を
pax headerとしても書く — `LIBARCHIVE.xattr.*` と `SCHILY.xattr.*` で、`tar -tzf` の一覧には出ず、
GNU tarは展開時に未知のキーワードとして警告する。翌日、同じプロジェクトで実測した（2026-09-23）:
`COPYFILE_DISABLE=1` の修正を出した後のarchiveにも `com.apple.provenance` が残り、
作業ツリーが同期フォルダーの下にあるため `com.dropbox.attrs` も入っていた。
stagingに通常ファイルだけを置いてもarchive内が同じとは限らず、直したはずの設定も証拠にならない。
逆も成り立つ: `COPYFILE_DISABLE=1` なしの `--no-xattrs` はpax headerを落とすが、`._` エントリは書く。
GNU tar 1.35で展開した結果（2026-09-23）: どちらもなし — 警告と余分な `._` ファイル。
`COPYFILE_DISABLE=1` のみ — 警告。`--no-xattrs` のみ — 警告なし、余分な `._` ファイルあり。両方 — 正常。

**適用方法:** `COPYFILE_DISABLE=1 tar --no-xattrs` でarchiveを作る — 2つの形を1つずつ止める
（`--no-xattrs` はbsdtar・GNU tarのどちらも受け付ける）。そのうえで**設定ではなくarchiveを検査する**:
リリースの関門でentry一覧を出し、配布契約（binary・README・license）と完全一致することを要求し、
`._*`・`PaxHeader`・`__MACOSX` のentryを拒否し、各entryの**pax header**に `LIBARCHIVE.xattr.` /
`SCHILY.xattr.` で始まるキーが無いことを確かめる（Pythonの `tarfile` の `pax_headers`）。
展開したストリームをgrepしてはいけない: ファイルの本文にも一致する。同梱したCHANGELOGがこの
キーワードを書いていたため、きれいなarchiveがリリースの関門で拒否された（2026-09-23）。一覧は `tar --options 'tar:!mac-ext' -tzf` で出す: macOSのbsdtarは
一覧表示のとき `._` エントリを対応するentryへ畳み込むため、素の `tar -tzf` はきれいな一覧を返し、
その上の `._` 検査は決して落ちない。並べた一覧はCロケールで比べる（`LC_ALL=C sort`）。
UTF-8ロケールでは `abuse-lookup` が `LICENSE` より前に並び、固定の期待文字列は正しいarchiveを拒否する。
関門は形ごとのfixture（`--no-xattrs` のみのものを含む）で実証し、fixtureは畳み込まない読み手
（Pythonの `tarfile`）で先に読み返す。どの関門でも落ちない断定を書いたことが、最初の修正が
効いたと信じられた理由である。署名・公証とarchive内容の検証はそれぞれ必要であり、片方の成功で他方を代用しない。

## C の安全策は、既定のままだと問題を見つけても失敗しない — 陽性対照は実際に動くビルド規則を読む

**事象:** timeout の 5 層（警告・静的解析・`-fbounds-safety`・ASan/UBSan・入力の揺さぶり）を Apple clang 21 で実測した。
- `clang --analyze -Werror` は指摘があっても rc=0。`-Xanalyzer -analyzer-werror` で初めて失敗する。
- UBSan は既定で報告して続行し、rc=0。`-fno-sanitize-recover=all` が要る。
- Apple の SDK は `_FORTIFY_SOURCE` を最大 2 として扱う（3 は効かない。`secure/_common.h` を読んで確認）。
- Apple clang に libFuzzer（`-fsanitize=fuzzer`）は無い。
- 層ごとの検出は重ならない。配列を越える `memcpy` は警告も静的解析も見逃し、ASan だけが捕まえた。長さ 3 の配列の
  添字 3 は `-fbounds-safety` が実行時に止め、通常ビルドは黙って読んだ。
- `-fbounds-safety` は、`main` の argv を `char *__null_terminated *__counted_by(argc)` で受ければ通り、`execvp` へ渡す
  1 か所だけ `__unsafe_forge_null_terminated` が要る。ローカル配列を `const char *` の引数へ渡す書き方と、`SIG_IGN` を
  関数ポインタの引数で渡す書き方は拒否される（`sa_handler` への直接代入、長さつきの `write` は通る）。
- 陽性対照がフラグの変数だけを見ていると、ビルド規則からフラグが外れても通る。`$(MAKE) -n` で規則を読んでも、
  `-f $(firstword $(MAKEFILE_LIST))` を付けないと、動いているのとは別の Makefile を読む。

**適用方法:** 層ごとに失敗させるフラグを明示する。わざとバグを入れたファイルで、各層が診断の文言つきで失敗することを
`make test` の最初に確かめる。フラグが規則に入っているかは、動いている Makefile の `make -n` の出力で確かめる。

## cgo のバイナリの最小 macOS は `MACOSX_DEPLOYMENT_TARGET` で決める — CGO_CFLAGS／CGO_LDFLAGS だけでは Go のランタイムがビルド機の OS のまま

**事象:** Core Text を cgo で呼ぶ MCP サーバ（report-studio-mcp）の最小 OS を 26.0 にしようとして、
`CGO_CFLAGS` と `CGO_LDFLAGS` に `-mmacosx-version-min=26.0` を渡した。ビルド機は macOS 27.2。
リンカは「より新しい macOS 向けに作られたオブジェクト」という警告を出した。Go のランタイムの cgo
オブジェクトが、ビルド機の OS 向けのまま作られていた（実測、2026-11-03）。

**なぜ:** フラグは自分のパッケージの C のコンパイルとリンクにしか届かない。Go のツールチェーンが
ランタイムの cgo 部分を作るときは、環境変数 `MACOSX_DEPLOYMENT_TARGET` を読む（観測した振る舞いからの推測。
ツールチェーンのソースは読んでいない）。

**適用方法:**
- Makefile で `export MACOSX_DEPLOYMENT_TARGET := <版>` とし、全体を作り直す。上の例では警告が 0 件になった。
- 確かめるのは `vtool -show-build <バイナリ>` の `minos`。期待した版であることを、リリースのたびに見る。
- この変数が効くのは、外部リンカが走るとき（いずれかのパッケージが `import "C"` するとき）だけ。
  cgo を使わない同じビルドは `minos 13.0` を報告した（実測）。
- Homebrew の formula の雛形は最小 macOS を宣言しない。古い OS では、入るのに起動しない。
  README に最小 OS を書く。

## vendored 依存へのローカルパッチは、適用・再ビルド・検査の 3 つを別々に確かめる — 入れ子の submodule では既定の仕組みがどれも足りない

**事象:** 画像生成 CLI（image-forge）が同梱する C++ ライブラリの、さらに内側の submodule に
1 関数のパッチを当てた。パッチをファイルとして持ち、ビルド前に当て、当たっていることを検査する
形にした。独立した検証が、その形の抜けを 3 つ見つけた（いずれも使い捨てのリポジトリと
`make -n` で実測、2026-11-13）。

1. **ライブラリを「パッチファイルより古ければ作り直す」にしていた。** submodule のポインタを
   動かしてもパッチファイルの時刻は変わらない。ライブラリは作り直されず、古いライブラリが
   そのままリンクされうる。
2. **検査が、パッチを当てたディレクトリだけを、インデックスと比べていた。** 外側のライブラリの
   ソースへの手編集、内側のチェックアウトでのローカルコミット、追跡されていないファイルは、
   どれも検査を通った。
3. **`.gitmodules` の `ignore = dirty` が、版数の `-dirty` も消していた。** パッチで作業コピーが
   常に固定コミットと異なるので、この設定は要る。しかし同じ設定で、同梱ツリーをどう編集しても
   `git describe --dirty` はきれいな版数を返す。

**なぜ:** 3 つとも「パッチを当てる」ことだけを見て仕組みを作ったためである。make のファイル時刻は
git のポインタを知らない。`git diff` は既定で作業ツリーとインデックスを比べる。`ignore = dirty` は
状態表示を静かにする設定で、何が静かになるかを選べない。

**適用方法:**
- 同梱ライブラリのビルドは毎回走らせ、何が古いかはそのライブラリ自身のビルドシステム（cmake など）に
  判断させる。最新のツリーなら数秒で済む。パッチの適用もその直前に毎回行う（当たっていれば何もしない）。
- 検査は、同梱したチェックアウトと、その中のすべての submodule を対象にする。それぞれについて、
  親が記録するコミットの上にあること（`git submodule status --recursive` の行頭）、追跡されて
  いないファイルが無いこと、固定コミットとの差分（`git diff HEAD --numstat`）がパッチの分と
  一致することを見る。パッチの無いディレクトリは差分が空であること。
- `ignore = dirty` を使ったら、版数文字列は同梱ツリーの状態を映さなくなったものとして扱い、
  上の検査をバイナリを作る経路に必ず通す。
- ポインタを動かす手順（パッチを外す → 動かす → ビルド）を、検査のスクリプトの冒頭に書く。
  パッチが当たったまま `git submodule update` すると、拒まれるか、編集が確かめられずに持ち越される。
- 検査が比べているものが名前と行数だけなら、そう書く。「手編集からはバイナリを作れない」と
  書いた文書は、検査の実装より強いことを言っていた。

## 依存の不具合を「渡す設定」で避けたら、設定の届かない要素が残る — 条件を 1 つだけ満たす入力を作って測る

**事象:** 同じライブラリを更新したところ、ある重み形式（1 バイトの FP8）を持つモデルで GPU
バックエンドが異常終了するようになった。ライブラリには「名前の規則に合うテンソルを読み込み時に
別の型へ変換する」設定があり、それで該当ファイルを変換させた。カタログのモデルはすべて動いた。
リリース前の独立検証が、ライブラリのソースから「一部のテンソルはどの規則からも除外される」ことを
読み取った。該当する実ファイルは手元に無かった。既存のチェックポイントのテンソル 1 つだけを
その形式に書き換えた複製を作ると、新版は異常終了し、公開中の旧版は生成できた（実測）。
回帰であることが確定し、設定での回避をやめて原因の関数にパッチを当てた。

これに先立ち、異常終了の理由そのものが見えなかった。このライブラリは、ログの受け取り口を登録しないと
バックエンドのエラーを含むすべてのログを捨てる。受け取り口を登録して初めて、欠けているカーネルの名前が
出た（実測）。

**なぜ:** 設定は、依存が用意した範囲にしか届かない。その範囲は依存のソースにあり、こちらの
テスト対象（カタログのモデル）がたまたま範囲内に収まっていると、動作確認は全部通る。

**適用方法:**
- 依存の不具合を設定で避けるときは、その設定が**適用されない条件**を依存のソースで読む。
  除外リスト、型による分岐、別経路。
- その条件に当たる実物が手元に無ければ、作る。既存の入力の 1 要素だけを書き換えた複製は、
  ヘッダを書き直して末尾にデータを足すだけで作れることが多い。作った入力は、新版と公開中の版の
  両方に通す。片方だけ落ちれば回帰である。
- 「クラッシュして理由が分からない」ときは、依存がログを捨てていないかを最初に確かめる。
  コールバックの登録が要るライブラリは、登録しなければ黙っている。
- 設定での回避が届かないと分かったら、原因の箇所を直す側（パッチ、上流への報告）に切り替える。
  届かない範囲を自分の側で列挙して拒否する案は、依存の除外リストの写しを持つことになる。
