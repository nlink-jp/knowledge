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
