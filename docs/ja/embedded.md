# 組み込み（M5Stack / ESP32）

M5Stack デバイス開発の知見。各項目は「事象 → なぜ → 適用方法」。

---

## 旧 M5Stack ライブラリから M5Unified への移行

**事象:** 旧ライブラリ（M5Stack.h / M5Core2.h）は ESP32 ボードパッケージ v3.x と非互換
（`rom/miniz.h` 欠落）。M5Unified が後継。

**API 対応表:**
- `M5.Lcd` → `M5.Display`
- `M5.begin(lcd,sd,serial,i2c)` → `auto cfg = M5.config(); M5.begin(cfg);`
- `TFT_eSprite(&M5.Lcd)` → `M5Canvas(&M5.Display)`
- `M5.Axp.GetBatVoltage()` → `M5.Power.getBatteryLevel()`（0–100 を直接返す）
- `M5.Rtc.SetTime/SetDate` → `M5.Rtc.setDateTime(m5::rtc_datetime_t)`
- `M5.IMU.Init()` → `M5.Imu`（自動検出）

**注意点:**
- ESP32 v3.x に `NetworkManager` クラスがある — 同名クラスを定義すると再定義エラー。
- M5Stack Basic v2.7 は **PSRAM なし** — フルスクリーン `createSprite(320,240)`（150KB）は
  確保失敗して画面が真っ暗になる。直接描画 + 背景色上書きで対応。Core2 は PSRAM ありで
  Sprite ダブルバッファ可。
- SD 初期化は自動で行われる場合とそうでない場合がある（Core2 は `SD.begin(4)` が必要）。
- `M5.begin()` が Serial2 を初期化する場合がある — GPS 等で Serial2 を使うなら
  `end()` → 再 `begin()`。
- **RTC の二重 offset 問題**: `configTime(offset,...)` + `getLocalTime()` + RTC 書込は
  offset が二重適用される。NTP は必ず `configTime(0,0,...)` で UTC 取得し、表示時のみ
  offset を適用する。

## M5 GPS Module v2.1（AT6668）はデフォルト 115200 baud

**事象:** ネット上の情報の多くは旧版 AT6558（9600 baud）前提。v2.1 の AT6668 を 9600 で
接続すると、バイナリに見える文字化けデータが流れる（= ボーレート不一致のサイン）。

**適用方法:**
- `Serial2.begin(115200, SERIAL_8N1, 16, 17)`（M5Stack Basic v2.7 で TXD=G17, RXD=G16、
  ディップスイッチでピン選択可）。
- コールドスタートは 5–15 分。fix 前でも GSV の in-view 衛星数は取得できる。
- TinyGPS++ の `satellites.value()` は GGA の fix 衛星数のみ。in-view は `TinyGPSCustom`
  で GSV フィールド 3 を取る。マルチコンステレーションでは GN プレフィックスが使われ、
  GSV は GP/BD/GL 別に流れる。

## M5Module-GNSS ライブラリは M5Unified と共存不可

**事象:** M5Module-GNSS（公式ライブラリ）は Bosch BMI270 API を内蔵しており、M5Unified の
IMU ドライバとシンボル競合して `M5.begin()` がハングする。旧 ESP-IDF i2c ドライバを使って
いるため ESP32 Core v3.x の新ドライバとも `abort()` で競合する。

**適用方法:** M5Module-GNSS は使わない。IMU は M5Unified 内蔵の `M5.Imu`（BMI270）を使い、
BMM150 は BMI270 の AUX レジスタパススルーで Wire 直接読み取り、BMP280 は独立ライブラリで
使う。

## ESP32 Arduino コアで `--output-dir` を使うと、スケッチのフォルダに絶対パス入りのビルド生成物が書かれる

**事象:** `arduino-cli compile --output-dir <dir>` で esp32:esp32 コア（3.3.8 で実測）の
スケッチをビルドすると、指定した `<dir>` とは別に `<スケッチ>/build/<fqbn>/` ができ、
`.bin`・`.elf`・`.map`・`sdkconfig`・`build.options.json` などがコピーされる。`.map` と
`build.options.json` にはビルド機の絶対パス（ホームディレクトリ）が入っている。
スケッチのフォルダはソースツリーの中なので、`git add` でそのままコミットに入った
（push 前にホームディレクトリ検査で発覚）。

**なぜ:** コアの `platform.txt` にある `recipe.hooks.savehex.postsavehex.*` が、バイナリを
書き出す（エクスポートする）たびに `{sketch_path}/build/` へコピーする。arduino-cli の
`sketch.always_export_binaries` が `false` でも止まらない — コピーしているのは
arduino-cli ではなくコア側のフックだから。

**適用方法:**
- `--output-dir` や `-e` を使わず、`--build-path <dist 配下の絶対パス>` でビルド
  ディレクトリそのものを成果物置き場にする。書き出しをしないのでフックは走らない。
  書き込みは `arduino-cli upload --input-dir <同じパス>`。
- ビルド後に `<スケッチ>/build` が存在したら失敗させる検査を Makefile に置く。
  `.gitignore` に足して隠すのではなく、作らない側で止める（成果物の置き場は `dist/` だけ、
  という規約とも合う）。

## arduino-cli の `--build-property` で文字列マクロを渡すときは、フラグ全体を単引用符で囲む

**事象:** 版数をファームウェアに埋め込もうとして、
- `compiler.cpp.extra_flags=-DFW_VERSION=\"v1\"` はバックスラッシュごと gcc に届き
  `stray '\' in program` になる。
- `compiler.cpp.extra_flags=-DFW_VERSION='"v1"'` は単引用符ごと届き、`'"v1"'` が
  複数文字の文字定数（`int`）になって `invalid conversion from 'int' to 'const char*'`
  になる。

**なぜ:** arduino-cli はレシピの文字列を自前で引数に分割する。引用符が特別なのは
**引数の先頭にあるときだけ**で、対応する引用符までを 1 つの引数にまとめる。引数の途中の
引用符とバックスラッシュは、そのまま渡される（arduino-cli 1.5.1 で実測）。

**適用方法:**
- フラグ全体を単引用符で囲む。Makefile なら
  `--build-property "compiler.cpp.extra_flags='-DFW_VERSION=\"$(VERSION)\"'"`
  （シェルの二重引用符の中で `\"` が `"` になり、arduino-cli には
  `'-DFW_VERSION="v1"'` が届く）。
- **フラグが複数あるときは 1 つずつ囲む。** `'-DA=0 -DB=0'` は 1 つの引数として届き、
  `A` が `0 -DB=0` と定義された（実測）。`'-DA=0' '-DB=0'` と書く。Makefile では
  `$(foreach d,$(DEFINES),'$(d)')` で並べる。
- `build.extra_flags` はボード定義が使っているので上書きしない。`platform.txt` が
  利用者向けに空で用意している `compiler.cpp.extra_flags` を使う。
- 効いたかは推測せず、`strings <.bin> | grep <版数>` で埋め込まれたことを確かめる。

## M5Stack BASIC v2.7 への書き込み・読み出しは、macOS では 230400 baud に落とす

**事象:** Arduino の `m5stack_core` ボード定義は、書き込み速度の既定値がメニュー先頭の
1500000 baud になっている。BASIC v2.7 の USB シリアル（WCH CH9102F、`1a86:55d4`、
macOS 標準ドライバ、Thunderbolt ハブ経由）で esptool を使うと、921600 baud では
`The chip stopped responding`、460800 baud では `Invalid head of packet` で途中停止した。
230400 baud では 4MB の読み出しも書き込みも通り、ハッシュ照合も一致した
（2026-09-24、esptool 5.2.0 で実測。この 1 構成での結果）。

**適用方法:**
- Makefile で `arduino-cli upload --board-options UploadSpeed=230400` のように速度を
  明示し、既定値に任せない。上げるのは実測してから。
- 書き込みでファームウェアを上書きする前に、`esptool read-flash` で控えを取る。
  まず `read-flash 0x8000 0xC00` でパーティション表を読み（`gen_esp32part.py` で
  読める）、使われている範囲だけを控えると速い（標準の 4MB 構成なら先頭 4MB）。
  控えには NVS（Wi-Fi 設定などが入り得る）も含まれるので、扱いに注意する。

## macOS に BLE キーボード（HID over GATT）として接続させるなら、HID の読み取りを暗号化必須にする

**事象:** ESP32（Arduino-ESP32 3.3.8 同梱の BLE ライブラリ）で HID キーボードを作り、ペアリングに
番号入力（Passkey Entry）を使った。macOS 27.0 では、ペアリングは成功して「キーボード」として接続
済みになるのに、キーがまったく効かなかった。macOS の入力デバイスの一覧（`hidutil list`）にも出ず、
機器を再起動しても自動では再接続されなかった。

**なぜ:** macOS で BLE の HID を受け持つ `BTLEServer` は、接続するとすぐにレポートマップなどを読む。
それらが暗号化なしで読めると、利用者が番号を入力している間に読み終え、
「Not creating HID device as the link is not encrypted」と記録して HID 機器を作らない。30 秒後に
サービスの初期化が時間切れになり、再試行しない。そのうえ、この機器を自動接続の一覧から外す。
`/usr/bin/log show --predicate 'process == "BTLEServer"'` で読める。

**適用方法:**
- HID サービスのレポートマップ（0x2A4B）、HID 情報（0x2A4A）、プロトコルモード（0x2A4E）の権限を
  `ESP_GATT_PERM_READ_ENC_MITM` にする。最初の読み取りが「認証が足りない」で失敗するので、macOS は
  ペアリングを先に済ませ、その 1 秒後に HID 機器を作った（2 回中 2 回）。
- 一度失敗した機器は、Mac の Bluetooth 設定から削除し、機器側のペアリング情報も消してから、
  ペアリングし直す。
- 一度 HID 機器ができた後は、機器を再起動すると 1〜2 秒で自動的に再接続・再暗号化された。

## macOS は BLE の HID 機器の接続条件を自分で決める — 周辺機器への書き込みは、受信中に通知を出し続けると速くなる

**事象:** macOS 27.0 が BLE キーボードとして接続した ESP32 に、アプリから応答なし書き込みで
データを送ると、毎秒約 0.28 KB しか出なかった。

**なぜ:** macOS は HID 機器の接続を、間隔 15 ms・スレーブレイテンシ 22・データ長 90 に設定する。
機器から別の値を求めても、1 秒以内に元へ戻した（5 回。初回ペアリング直後に 1 度だけ、
レイテンシ 0 のままにしたことがある）。レイテンシ 22 では機器は約 345 ms ごとにしか受信しない。
受信中の機器が接続間隔ごと（15 ms）に通知を出すと、毎秒 4.1〜4.2 KB になった（30 ms ごとなら 2.5、
60 ms ごとなら 1.6）。周辺機器は、送るものがない間だけ接続の機会を飛ばせる、というのがこちらの読み
（規格の条文は確認していない）。

**適用方法:**
- 接続条件の変更要求は、当てにしない。見積もりはレイテンシ 22 で立てる。
- 大量に受け取る間は、機器が 15 ms ごとに状態の特性を通知し、ホストはそれを購読する。
- 完了は通知と読み取りの両方で確かめる。最後の通知を取りこぼしたことがある（10 回中 3 回）。
- ホスト側に書き込みを大量にためない。約 30 秒分の書き込みがたまった状態で読み取りを出すと、
  応答がないまま約 30.5 秒後に接続ごと切れた。ATT 要求の 30 秒の時間切れという仮説。
  機器の通知に合わせて、少しずつ送る。
- 計測では、接続条件を接続の時点から記録し、機器からは何も求めない版で測る。途中で条件が
  変わったことに気づかないと、結果の説明がつかなくなる（最初の計測でこれが起きた）。

## CoreBluetooth のアプリは、システムがキーボードとして接続中の BLE 機器の独自サービスを使える

**事象:** 機器の BLE 接続は、macOS 自身が HID キーボードとして持っている。その同じ機器にある
独自の GATT サービスを、アプリから読み書きしたかった。

**なぜ:** `retrieveConnectedPeripherals(withServices: [独自サービスの UUID])` が、システムが接続中の
その機器を返した。そこに `connect` すると、システムの接続を共有したまま、MITM 付きで暗号化された
独自の特性を読み書きできた（42 回中 42 回、macOS 27.0）。HID サービス（0x1812）はアプリからは
見えず、`withServices: [0x1812]` では何も返らない。

**適用方法:**
- 独自サービスの UUID で探す。0x1812 では見つからない。
- ペアリングは HID の時点で済んでいるので、アプリ側で 2 度目のペアリングは起きない。
  アプリに要るのは Bluetooth の利用許可だけ。許可はアプリ単位なので、CLI のような裸の
  バイナリではなく .app として動かす。

## ペアリング済みの Mac は GATT の構成を覚えている — Bluedroid は Service Changed を送れなかった

**事象:** ファームウェアを書き換えて、独自の特性に「通知」を足した。ペアリング済みの Mac からは、
古い構成のまま（読み取りのみ）に見えた。

**なぜ:** 接続のたびに構成を調べ直すのではなく、覚えた構成を使っている。構成の変更を知らせる
Service Changed を機器から送ろうとしたが、Arduino-ESP32 3.3.8 の Bluedroid は状態 0x82 で
送信そのものを拒んだ（8 回中 8 回）。Bluedroid の AUTO モードが送るのは、接続中にサービスを
足し引きしたときだけ、と読んでいる。機器側のスタックが送らないので、macOS がそれを受けて
どうするかは確かめられていない。

**適用方法:**
- 独自サービスの構成は、HID の定義と同じく、リリースごとに固定する。変えたら互換性を壊す変更として扱い、
  再ペアリング（Mac から削除し、機器のペアリング情報も消す）を求める。
- 機器にプロトコルの版を報告させ、アプリ側で食い違いを検出して、再ペアリングを案内する。

## Arduino-ESP32 で BLE だけを使うなら、先に Classic 用のメモリを解放する

**事象:** PSRAM のない ESP32（M5Stack BASIC v2.7）で、BLE の HID と独自サービスと画面描画を
同時に載せると、空きヒープが心配になる。

**なぜ:** Arduino-ESP32 3.3.8 の ESP32 向けビルド済み設定では Classic Bluetooth が有効で、
`BLEDevice::init()` はコントローラを BTDM（Classic と BLE の両方）で起動する。

**適用方法:**
- `BLEDevice::init()` の前に、`esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)` と
  `btStartMode(BT_MODE_BLE)` を呼ぶ。`BLEDevice::init()` は、すでに起動しているコントローラを
  そのまま使う。空きヒープは 21,036 B 増えた（広告開始後 92,084 → 113,120 B）。
- それでも、画面全体の圧縮画像（67 KB）を丸ごとメモリに溜めると、空きは 22,748 B まで下がった。
  大きなデータは届いたそばから展開する。
