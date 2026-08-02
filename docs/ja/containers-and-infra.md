# コンテナ・インフラ

macOS 上の Podman、コンテナ内のデータベース・描画ライブラリに関する知見。
各項目は「事象 → なぜ → 適用方法」。

---

## macOS + Podman Machine はネイティブ Linux コンテナと大きく異なる

**事象:** コンテナ化ツールの実機テストで複数の問題が発生。すべて VM 層
（Apple Hypervisor + virtiofs + gvproxy）に起因していた。

**適用方法（チェックリスト）:**
- **Unix ドメインソケットは virtiofs を通過できない**（ホストのエージェント類のソケットを
  コンテナに渡せない）。
- gvproxy がポートを保持し続けることがある — コンテナ停止後もポートが解放されない場合が
  ある。
- ビルド時 OOM: Podman Machine のデフォルトメモリ（~4GB）では不足する場合がある。8GB 推奨。
- コンテナ内サービスは **`0.0.0.0` バインド必須**（`127.0.0.1` だと VM 外からアクセス不可）。
- sshd は Dockerfile の `ENV` を子プロセスに渡さない — PATH 等は `.profile`/`.bashrc` に書く。
- macOS の `sed -i` は Linux と構文が異なる — `sed + mv` パターンを使う。
- macOS の `date -j -f` はタイムゾーン情報を無視する — epoch 値を直接取得する。

## DuckDB ファイルは pre-touch せず、親ディレクトリだけ bind-mount する

**事象:** ホスト側で空ファイルを touch してから bind-mount すると、DuckDB が
`IOException: not a valid DuckDB database file` で接続失敗する。

**なぜ:** DuckDB は 0 バイトのファイルを「空か壊れた DB か判別できない」ため safe に
エラー化する。

**適用方法:** DB ファイルは作業ディレクトリ内に配置する設計にし、**親ディレクトリだけ**を
`-v <host>/work:/work` でマウントする。DB ファイルは初回接続時に DuckDB に作らせる。
事前 touch コードは書かない。

## matplotlib（Agg）は font.sans-serif の先頭フォントで全文字を描く

**事象:** `font.sans-serif: DejaVu Sans, Noto Sans CJK JP, ...` の設定で日本語ラベルが
豆腐になり、`Glyph ... missing from font(s) DejaVu Sans` の警告が出た。

**なぜ:** matplotlib の Agg バックエンドは**最初に見つかった有効フォントで確定**し、
per-glyph fallback（字種ごとに別フォントを試す）が無い。

**適用方法:** CJK を描くなら CJK フォントを必ず先頭に:
`font.sans-serif: Noto Sans CJK JP, DejaVu Sans, ...`。
Noto Sans CJK JP は Latin/数字/記号も覆うので、先頭に置いても英語ラベルへの副作用はない。
検証は `python -W error::UserWarning` で日本語タイトルの savefig が警告なく通ることを確認。
