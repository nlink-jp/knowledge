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
## rsyslog の logrotate 設定は別パッケージ — 無いと一部のログだけが静かに肥大する

**事象:** 稼働 2.7 年のサーバで `/var/log/messages` が 62 MB・606,000 行に達し、一度も
ローテートされていなかった。`logrotate` 自体は毎日正常終了しており、`dnf`・`btmp`・`wtmp`
などのログは正しくローテートされていた。

**なぜ:** 新しい rsyslog は logrotate 設定を **`rsyslog-logrotate` サブパッケージに分離**
している。これが入っていないと `/etc/logrotate.d/rsyslog` が存在せず、rsyslog が書く
`messages`・`cron`・`secure`・`maillog`・`spooler` の 5 つだけがローテート対象から漏れる。
他のログは正常に回るので、`logrotate` の故障には見えない。

**適用方法:**
- 確認は `ls /etc/logrotate.d/rsyslog`。無ければ `dnf install rsyslog-logrotate`。
- 症状の探し方は `ls -la /var/log/` で、**ローテート済みの兄弟ファイル（`.1` や
  `-YYYYMMDD`）を持たない大きなログ**を見つけること。
- 肥大したログは日次のログ要約レポートを汚染する（testing の「ログから時刻の結論を出す前に
  素性を確定させる」を参照）。
- 導入しても初回ローテートは `weekly` 設定だと最大 1 週間発火しない。すぐ切り替えたい場合は
  強制する。切り替え前に長期分を退避するなら、要約ツールの走査対象外（`/var/log` の外）に
  置くこと。

## `logrotate -f` に個別の設定ファイルを渡すとグローバル設定が失われる

**事象:** `logrotate -f /etc/logrotate.d/rsyslog` で強制ローテートしたところ、`rotate 4`
が効かず、ローテート後のファイルが保持されずに削除された。

**なぜ:** logrotate は**渡されたファイルだけ**を読む。`/etc/logrotate.conf` を渡さない限り
そこに書かれたグローバル設定（`weekly`・`rotate 4`・`create`・`dateext`）は適用されず、
組み込みの既定（`rotate 0` = 保持しない）になる。

**適用方法:** 手動実行でも `/etc/logrotate.conf` を渡す:

```
logrotate -d -f /etc/logrotate.conf     # ドライラン。-d は何も変更しない
logrotate -f /etc/logrotate.conf
```

ドライランの出力で `rotating pattern: ... (N rotations)` を確認してから本番を実行する。
`(no old logs will be kept)` と出ていたら `rotate` が効いていない。systemd の日次実行は
`ExecStart=/usr/sbin/logrotate /etc/logrotate.conf` なので自動実行側は正しく、この罠は
手動実行だけで踏む。

## SSH の死活監視は、OpenSSH 9.8 以降では監視元自身を締め出す

**事象:** サーバの SSH が数分おきに不通になる現象を調査していた。`ping` と DNS は 100%
応答するのに SSH だけが不通で、ストレージの I/O ハングを疑った。原因は**調査に使っていた
到達性監視とリトライループそのもの**だった。

**なぜ:** OpenSSH 9.8 で追加され 9.9 で既定有効になった `PerSourcePenalties` は、送信元
アドレスごとにペナルティを累積する。既定値:

```
crash:90 authfail:5 noauth:1 grace-exceeded:10
refuseconnection:10 max:600 min:15
```

`nc -z` によるポート監視は「認証せずに切断」＝ `noauth`、タイムアウトした `ssh` は
`grace-exceeded`、強制切断は `crash` を積む。リトライを繰り返すと `max` に張り付き、
**送信元が最大 10 分間遮断される**。

**適用方法:**
- **ストレージの I/O ハングと紛らわしい。** 見分けは sshd 側のログで行う。ペナルティ中は
  接続の記録が一切残らない（パケットが sshd に届かない）。ディスク起因なら
  `Timeout before authentication` などが残る。
- SSH を死活監視の対象にするなら、監視元を `PerSourcePenaltyExemptList` に登録する。
- 障害調査中にリトライループを回すと**調査自体が症状を作る**。接続が落ちたら間隔を空ける。
  ペナルティは累積するので、間を置かない再試行は状況を悪化させるだけになる。
