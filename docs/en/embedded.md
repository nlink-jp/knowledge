# Embedded (M5Stack / ESP32)

M5Stack device development lessons. Each entry follows
**symptom → why → how to apply**.

---

## Migrating from legacy M5Stack libraries to M5Unified

**Symptom:** The legacy libraries (M5Stack.h / M5Core2.h) are incompatible with
ESP32 board package v3.x (missing `rom/miniz.h`). M5Unified is the successor.

**API mapping:**
- `M5.Lcd` → `M5.Display`
- `M5.begin(lcd,sd,serial,i2c)` → `auto cfg = M5.config(); M5.begin(cfg);`
- `TFT_eSprite(&M5.Lcd)` → `M5Canvas(&M5.Display)`
- `M5.Axp.GetBatVoltage()` → `M5.Power.getBatteryLevel()` (returns 0–100
  directly)
- `M5.Rtc.SetTime/SetDate` → `M5.Rtc.setDateTime(m5::rtc_datetime_t)`
- `M5.IMU.Init()` → `M5.Imu` (auto-detected)

**Caveats:**
- ESP32 v3.x has a `NetworkManager` class — defining one of your own causes
  redefinition errors.
- M5Stack Basic v2.7 has **no PSRAM** — a full-screen `createSprite(320,240)`
  (150 KB) fails allocation and the screen goes black. Draw directly with
  background-color overwrites. Core2 has PSRAM and can double-buffer sprites.
- SD init is sometimes automatic, sometimes not (Core2 needs `SD.begin(4)`).
- `M5.begin()` may initialize Serial2 — for GPS etc., `end()` then re-`begin()`.
- **RTC double-offset problem**: `configTime(offset,...)` + `getLocalTime()` +
  RTC writes apply the offset twice. Always fetch NTP as UTC via
  `configTime(0,0,...)` and apply the offset at display time only.

## M5 GPS Module v2.1 (AT6668) defaults to 115200 baud

**Symptom:** Most online material assumes the older AT6558 (9600 baud).
Connecting the v2.1 AT6668 at 9600 yields binary-looking garbage (= the
baud-mismatch tell).

**How to apply:**
- `Serial2.begin(115200, SERIAL_8N1, 16, 17)` (TXD=G17, RXD=G16 on M5Stack Basic
  v2.7; DIP switches select pins).
- Cold start takes 5–15 minutes. In-view satellite counts (GSV) are available
  before a fix.
- TinyGPS++'s `satellites.value()` is GGA fix satellites only; get in-view via
  `TinyGPSCustom` on GSV field 3. Multi-constellation uses the GN prefix, with
  GSV split per GP/BD/GL.

## The M5Module-GNSS library cannot coexist with M5Unified

**Symptom:** M5Module-GNSS (official) embeds the Bosch BMI270 API, which
symbol-clashes with M5Unified's IMU driver and hangs `M5.begin()`. It also uses
the legacy ESP-IDF i2c driver, which `abort()`s against ESP32 Core v3.x's new
driver.

**How to apply:** Don't use M5Module-GNSS. Use M5Unified's built-in `M5.Imu`
(BMI270); read BMM150 via BMI270's AUX register passthrough over Wire; use
BMP280 with its standalone library.
