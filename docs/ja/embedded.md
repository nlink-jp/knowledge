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
