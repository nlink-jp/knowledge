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

## macOS のボリュームを case-sensitive でフォーマットしない — Apple 純正アプリが静かに壊れる

**事象:** 外付け NVMe（case-sensitive APFS）に写真ライブラリを置いていたところ、
`photolibraryd` が CPU 100% を数時間〜数十時間占有し続けた。写真 1.9 万枚のライブラリで
DB が 6 GB に肥大し、顔検出レコード 21 万件のうち **98% が「参照先の写真が存在しない」
孤児**になっていた。ライブラリ修復・人物データの再構築・ライブラリの完全な作り直しを
順に試したがいずれも無効で、**原因特定まで 2 日**かかった。同じデータを同期している
他のデバイスはすべて正常に動作していた。

**なぜ:** Apple は写真ライブラリを case-sensitive ボリュームに置くことを非対応と明記して
いる（「ファイルが見つからなくなるためデータ損失を招く可能性がある」）。APFS は既定で
case-insensitive だが、**フォーマット時に case-sensitive を選べてしまう**のが罠。

macOS は歴史的に case-insensitive が前提で、対応していないソフトウェアは珍しくない。
Adobe Creative Cloud は case-sensitive ボリュームへのインストールを拒否し、Steam は
起動時にエラーを出す。**拒否してくれるアプリは幸運な部類**で、Apple 純正アプリは拒否せず
静かに壊れる。今回観測した連鎖:

```
case-sensitive ボリューム
  → アセットの参照解決が失敗（ログに "unexpectedly has no asset" が毎分数百件）
  → 参照先を失ったレコードが孤児として蓄積
  → 孤児から派生レコードが生成され続けて数十万件に膨張
  → OS が重複排除を試み、削除の連鎖と保存時の追加フェッチで O(n²) 化
  → デーモンが CPU を張り付かせたまま、ディスクには何もコミットされない
```

**適用方法:**
- **ライブラリ類（写真・音楽・その他 Apple 純正アプリのデータ）を外付けに置く前に確認する。**

  ```sh
  diskutil info /Volumes/<name> | grep 'File System Personality'
  # "APFS" = OK / "Case-sensitive APFS" = NG
  ```

  実測で確かめるなら、大文字違いのファイルが同一視されるかを見る。

  ```sh
  touch /Volumes/<name>/.ctest_a
  [ -f /Volumes/<name>/.ctest_A ] && echo OK || echo "NG (case-sensitive)"
  rm -f /Volumes/<name>/.ctest_a
  ```

- **新規フォーマット時に case-sensitive を選ばない。** 必要な理由が明確にある場合を除き、
  既定の case-insensitive を使う。case-sensitive が要るのは Linux 由来のソースツリーを
  そのまま扱う場合などに限られ、メディア保管には不要。
- **他のデバイスが正常でも安心材料にならない。** 壊れたデータはクラウド同期で他デバイスへ
  伝播しうるが、取り込みを終えた機体は無症状のまま動き続ける。UI にも現れないため、
  異常に気づけるのは DB を直接数えたときだけ。
- **部分対処で済ませない。** 1 つのライブラリを退避しても、同じボリュームに載っている
  他の Apple 純正ライブラリは同じ違反を続けている。ボリューム構成ごと見直す。
- **移行前に case 衝突を検査する。** case-insensitive へ移すと、大文字小文字だけが異なる
  ファイルは衝突して失われる。パスを小文字化して重複を数え、0 件を確認してから移す。
- **切り分けの一般則として:** 同じデータを共有する複数台のうち 1 台だけが壊れるなら、
  原因はデータではなくその 1 台の固有条件にある。**ファイルシステムのフォーマットを
  チェック項目に必ず含める。** 症状を掘る前に「他の環境では起きるか」を確認すると、
  探索範囲が一気に絞れる。

## launchd の周期実行は遅延・スキップし得るし、それを観測する手段もない

**事象:** 30 分周期の launchd ジョブ（AI エージェントのバッチ分析起動）が 1 回分
スキップされ、1 時間後に実行された。未処理データが増えてターン実行時間と
トークン消費が膨張する正帰還になった。AI による事後調査でも原因は特定できなかった。

**なぜ:** OS X 10.9 以降、launchd のタイマーは省電力のため coalescing される
（発火が遅延し得る）。さらに同一ラベルのジョブが実行中だと次の発火は**捨てられる**
（キューされない）。どちらが起きたかを事後に知る手段がない — launchd は次回発火
時刻を照会できず、スキップの記録も残さない。unified log の保持期間（数日程度）を
過ぎれば追跡も不可能になる。

**適用方法:**
- plist の `LegacyTimers: true` で coalescing を無効化できる（精密発火になるが
  省電力性は犠牲）。
- ジョブの実行時間が周期を超え得る設計なら、launchd の周期実行を使わない —
  実行中に来た発火は黙って捨てられる。
- 「実行されなかったこと」の観測が必要な周期タスクは
  [task-clock](https://github.com/nlink-jp/task-clock) を使う: cron 式を自前評価し、
  全発火を予定 vs 実績（`on_time` / `queued` / `missed` + 理由）で記録する。
  launchd は KeepAlive による常駐維持のみに使う。
- 常駐デーモンの plist は `ProcessType: Interactive` にして background 絞り込みを
  回避する。タイミングループは「次回まで眠る長い timer 一本」ではなく短周期 ticker
  のポーリングにする — App Nap / coalescing 下でも最悪遅延が ticker 間隔に有界化される。

## launchd デーモンのバイナリを他者のライフサイクルに置かない

**事象:** GUI アプリ同梱の CLI から `install` したデーモンの plist が
`os.Executable()`（= .app 内部パス）を指していた。アプリを終了しただけで
デーモンが道連れになり、実行中タスクが kill されて作業が失われた。

**なぜ:** launchd はプログラムパスの実体が消えれば spawn できず、実行中でも
親アプリの終了処理・cask upgrade（.app 差し替え）・`brew upgrade`（Cellar 削除）・
再ビルド（dist/ 再署名）がバイナリを壊す。plist に書いたパスの寿命 = デーモンの
寿命になる。

**適用方法:**
- install はバイナリを**デーモン専用のホーム**（例: `data_dir/bin/<name>`）へ
  自己コピーし、plist はそこだけを指す。順序は bootout → copy → plist 書き込み →
  bootstrap（実行中バイナリの差し替えは署名変更で SIGKILL される・copy 失敗時に
  実体のないパスを指す plist を残さない）。
- 停止の永続化は `launchctl disable` + `bootout`。disable なしの bootout は
  次回ログインの RunAtLoad で意図に反して蘇る。disable されたサービスの
  bootstrap は拒否されるため、起動側は必ず `enable` を先に打つ。
- デーモン停止で子プロセスの作業を失わせない設計は、**spawn 時に**
  pid + プロセス開始時刻（`ps -o lstart=`）をレジストリへ記録し、次のデーモンが
  生存 + 開始時刻一致を確認して引き取る（graceful 停止専用の停止時記録は
  クラッシュを守れない。コマンド列照合は run 内 exec で偽陰性・pid 再利用で
  偽陽性になる — 開始時刻は exec を跨いで不変かつ pid 再利用で必ず変わる）。
  実装例: [task-clock](https://github.com/nlink-jp/task-clock) の live-run registry。
