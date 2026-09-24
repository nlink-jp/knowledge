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

## With the ESP32 Arduino core, `--output-dir` writes build output with absolute paths into the sketch folder

**Symptom:** Building a sketch for the esp32:esp32 core (measured on 3.3.8) with
`arduino-cli compile --output-dir <dir>` also creates `<sketch>/build/<fqbn>/`,
separate from `<dir>`, and copies the `.bin`, `.elf`, `.map`, `sdkconfig`,
`build.options.json` and more into it. The `.map` and `build.options.json` hold
the build machine's absolute paths (its home directory). The sketch folder is in
the source tree, so `git add` took them straight into a commit (caught before
push by a home-directory check).

**Why:** `recipe.hooks.savehex.postsavehex.*` in the core's `platform.txt` copy
the output into `{sketch_path}/build/` every time binaries are exported.
arduino-cli's `sketch.always_export_binaries` set to `false` does not stop it —
the copy is made by the core's hooks, not by arduino-cli.

**How to apply:**
- Do not use `--output-dir` or `-e`. Make the build directory itself the
  artifact directory with `--build-path <absolute path under dist>`; nothing is
  exported, so the hooks never run. Flash with
  `arduino-cli upload --input-dir <the same path>`.
- Put a check in the Makefile that fails the build when `<sketch>/build` exists.
  Stop it being created rather than hiding it in `.gitignore` (which also keeps
  to the convention that `dist/` is the only place for build output).

## To pass a string macro through arduino-cli's `--build-property`, wrap the whole flag in single quotes

**Symptom:** Embedding a version in the firmware,
- `compiler.cpp.extra_flags=-DFW_VERSION=\"v1\"` reaches gcc with the
  backslashes and fails with `stray '\' in program`;
- `compiler.cpp.extra_flags=-DFW_VERSION='"v1"'` reaches it with the single
  quotes, `'"v1"'` becomes a multi-character constant (an `int`), and the build
  fails with `invalid conversion from 'int' to 'const char*'`.

**Why:** arduino-cli splits the recipe string into arguments itself. A quote is
special **only as the first character of an argument**, where it groups
everything up to the matching quote into one argument. Quotes and backslashes in
the middle of an argument are passed on as they are (measured with
arduino-cli 1.5.1).

**How to apply:**
- Wrap the whole flag in single quotes. In a Makefile:
  `--build-property "compiler.cpp.extra_flags='-DFW_VERSION=\"$(VERSION)\"'"`
  (inside the shell's double quotes `\"` becomes `"`, so arduino-cli receives
  `'-DFW_VERSION="v1"'`).
- Leave `build.extra_flags` alone — the board definitions use it. Use
  `compiler.cpp.extra_flags`, which `platform.txt` leaves empty for the user.
- Do not assume it worked: confirm the version is in the binary with
  `strings <.bin> | grep <version>`.

## On macOS, flash and read an M5Stack BASIC v2.7 at 230400 baud

**Symptom:** The Arduino `m5stack_core` board definition's default upload speed
is its first menu entry, 1500000 baud. With the BASIC v2.7's USB serial chip
(WCH CH9102F, `1a86:55d4`, macOS's own driver, through a Thunderbolt hub),
esptool stopped part-way at 921600 baud with `The chip stopped responding` and
at 460800 baud with `Invalid head of packet`. At 230400 baud a 4 MB read and a
firmware write both completed with matching hashes (measured 2026-09-24 with
esptool 5.2.0, on this one setup).

**How to apply:**
- State the speed in the Makefile, e.g.
  `arduino-cli upload --board-options UploadSpeed=230400`, instead of relying on
  the default. Raise it only after measuring.
- Before overwriting a device's firmware, take a copy with `esptool read-flash`.
  Read the partition table first with `read-flash 0x8000 0xC00` (decode it with
  `gen_esp32part.py`) and copy only the range in use — the first 4 MB for the
  standard 4 MB layout — which is much faster. The copy includes NVS, which can
  hold Wi-Fi settings; handle it accordingly.
