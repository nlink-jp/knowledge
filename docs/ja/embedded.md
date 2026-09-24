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
