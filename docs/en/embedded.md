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
- **With several flags, quote each one.** `'-DA=0 -DB=0'` arrives as a single
  argument, and `A` was defined as `0 -DB=0` (measured). Write `'-DA=0' '-DB=0'`;
  in a Makefile, `$(foreach d,$(DEFINES),'$(d)')`.
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

## To be a BLE keyboard (HID over GATT) on macOS, make the HID reads require encryption

**Symptom:** An ESP32 HID keyboard (the BLE library bundled with Arduino-ESP32
3.3.8), paired with Passkey Entry. On macOS 27.0 the pairing succeeded and the
device showed as a connected keyboard, but no key did anything. It was not in
macOS's list of input devices (`hidutil list`), and after a reboot of the device
the Mac did not reconnect on its own.

**Why:** `BTLEServer`, macOS's BLE HID host, reads the report map and the rest
as soon as it connects. If they are readable without encryption, it finishes
while the user is still typing the passkey, logs "Not creating HID device as the
link is not encrypted", times the service out 30 s later without retrying, and
drops the device from its auto-connect list. Read it with
`/usr/bin/log show --predicate 'process == "BTLEServer"'`.

**How to apply:**
- Give the HID service's report map (0x2A4B), HID information (0x2A4A) and
  protocol mode (0x2A4E) the permission `ESP_GATT_PERM_READ_ENC_MITM`. The first
  read then fails for insufficient authentication, macOS pairs first, and it
  built the HID device one second after pairing (2 of 2).
- A device that already failed must be removed in the Mac's Bluetooth settings,
  its own bond erased, and paired again.
- Once the HID device existed, a reboot of the device was followed by an
  automatic reconnection and re-encryption within 1–2 s.

## macOS sets the link of a BLE HID device itself — writes to the peripheral go faster when it notifies while receiving

**Symptom:** Sending data from an app with write-without-response to an ESP32
that macOS 27.0 held as a BLE keyboard gave about 0.28 KB/s.

**Why:** macOS sets an HID device's link to a 15 ms interval, slave latency 22
and data length 90. When the device asked for other values, macOS put them back
within a second (5 times; once, right after a first pairing, it kept latency 0).
At latency 22 the device listens only about every 345 ms. With the receiving
device notifying every connection interval (15 ms), the rate was 4.1–4.2 KB/s
(every 30 ms: 2.5, every 60 ms: 1.6). Our reading — not a cited clause of the
specification — is that a peripheral skips connection events only while it has
nothing to send.

**How to apply:**
- Do not rely on requests to change the link; budget at latency 22.
- While receiving a lot, have the device notify a status characteristic every
  15 ms, subscribed to by the host.
- Confirm completion by the notification and by a read: the final notification
  was missed (3 of 10).
- Do not let the host queue a lot of writes. With about 30 s of writes queued, a
  read got no answer and the link dropped about 30.5 s later — hypothesis: the
  30 s ATT transaction timeout. Send in small amounts, paced by the device's
  notifications.
- When measuring, log the link parameters from the moment of connection and
  measure with a device that asks for nothing. A change of parameters that goes
  unnoticed makes the results unexplainable (it happened in the first attempt).

## A CoreBluetooth app can use a custom service of a BLE device the system holds as a keyboard

**Symptom:** macOS itself holds the device's BLE connection as an HID keyboard;
an app needed to read and write a custom GATT service on the same device.

**Why:** `retrieveConnectedPeripherals(withServices: [custom service UUID])`
returned the system-connected device. After `connect`, the app shared the
system's link and read and wrote the MITM-encrypted custom characteristics (42 of
42, macOS 27.0). The HID service (0x1812) is hidden from apps:
`withServices: [0x1812]` returns nothing.

**How to apply:**
- Look it up by the custom service's UUID, not 0x1812.
- The pairing happened for HID, so the app causes no second pairing; it needs
  only the Bluetooth permission. That permission is per app, so run the code as
  an .app, not as a bare CLI binary.

## A bonded Mac remembers the GATT layout — Bluedroid did not send Service Changed

**Symptom:** A firmware update added a notify property to a custom
characteristic; the bonded Mac still saw the old layout (read only).

**Why:** The Mac uses the layout it learned rather than rediscovering on every
connection. The device tried to send Service Changed, but Bluedroid in
Arduino-ESP32 3.3.8 refused to send it with status 0x82 (8 of 8). Our reading is
that its AUTO mode sends it only for services added or removed while connected.
Since the device's stack never sent it, what macOS would do with one is unknown.

**How to apply:**
- Fix the custom service's layout per release, like the HID descriptor. A change
  is breaking and requires re-pairing (remove the device on the Mac and erase the
  device's bond).
- Fix the whole attribute table, handles included, not just the UUIDs. The
  bundled library stores subscriptions (CCCDs) in NVS by peer and handle, so a
  core update or a refactor that shifts handles is enough for a mismatch. Create
  the services in a fixed order with fixed handle reservations, print the table
  at boot, and compare it on the device with a pinned copy.
- Keep a *layout version* apart from the protocol version, since they break
  differently. The app checks the UUIDs and properties of the characteristics it
  sees as well as the version in Info (a stale cache can land a read on another
  attribute).
- On the device, store the layout version beside the bond; at boot, if it
  differs — **or is missing** — erase every bond, then store the version. Store
  it even when there was no bond: otherwise a fresh device erases its first bond
  on its second boot. Removal completes asynchronously; wait for the list to
  empty.

## With Arduino-ESP32 and BLE only, release the Classic controller memory first

**Symptom:** On an ESP32 without PSRAM (M5Stack BASIC v2.7), BLE HID, a custom
service and screen drawing together leave little heap.

**Why:** The prebuilt ESP32 configuration of Arduino-ESP32 3.3.8 enables Classic
Bluetooth, and `BLEDevice::init()` starts the controller in BTDM mode (Classic
and BLE).

**How to apply:**
- Before `BLEDevice::init()`, call `esp_bt_controller_mem_release(ESP_BT_MODE_CLASSIC_BT)`
  and `btStartMode(BT_MODE_BLE)`; `BLEDevice::init()` uses the controller already
  running. Free heap rose by 21,036 B (after advertising: 92,084 → 113,120 B).
- Holding a full-screen compressed image (67 KB) in RAM still took free heap
  down to 22,748 B. Decode large payloads as they arrive.

## The Arduino-ESP32 BLE library's `notify()` sends to every connected peer, without checking encryption

**Symptom:** With a BLE keyboard (HID over GATT) that sends input reports with
`notify()`, a button pressed while an unpaired peer is connected can send the
key to that peer too.

**Why:** In Arduino-ESP32 3.3.8, `BLECharacteristic::notify()` calls
`esp_ble_gatts_send_indicate` for everyone in `getPeerDevices()`, without
checking each peer's encryption or subscription (read in the source). Requiring
encryption to read a characteristic does not apply to its notifications. Whether
the stack itself holds them back on an unencrypted link was not checked.

**How to apply:**
- Send anything private (key reports) with `esp_ble_gatts_send_indicate` to the
  connection encrypted with the bond, by its connection id, not with `notify()`.
- Count a connection as encrypted from a successful authentication-complete
  event for the bonded address until it disconnects. The library restarts
  advertising only after a disconnect, so there is one connection at a time.

## The Arduino-ESP32 BLE library stores every CCCD write in NVS, whoever wrote it

**Symptom:** A peer that never pairs still adds NVS entries by writing a
subscription (CCCD).

**Why:** In 3.3.8, `BLEDescriptor` calls `BLE2902::persistValue()` on every
write to a 0x2902 descriptor. Its comment says "for bonded devices", but it
checks neither bonding nor encryption. The key is the last 2 bytes of the peer's
address plus the handle (read in the source). `BLEHIDDevice` makes the input
report and battery CCCDs writable without encryption. A peer that reconnects
with new addresses can fill the NVS that holds the bonds and the app's settings
(an inference, not tried on a device). The stored subscriptions also survive a
bond erase: `BLE2902::deleteAllPersistedValues()` exists, but the library never
calls it.

**How to apply:**
- Make every CCCD require encryption to write
  (`ESP_GATT_PERM_READ | ESP_GATT_PERM_WRITE_ENC_MITM`). With the HID CCCDs closed
  too, macOS 27.0 paired, set up the keyboard and received keyboard and consumer
  reports (measured 2026-09-24; the library's comment that HID enumeration needs
  them open did not hold on macOS).
- When erasing a bond, always call `deleteAllPersistedValues()` too.

## Bluedroid deletes the bond of a peer whose pairing or encryption fails

**Symptom:** Even on a bonded device, a failed pairing or encryption with a
peer can remove that peer's bond.

**Why:** The authentication-complete handler in the `libbt.a` bundled with
Arduino-ESP32 3.3.8 removes the peer's bond when the failure reason is anything
but "pairing not supported" (read from its disassembly, not from source). A
nearby device that stages a failing pairing with the bonded Mac's address could
make the device drop the bond (an inference).

**How to apply:**
- Never accept pairing because a bond is missing. Open a pairing window only for
  a cause recorded by a user's act (a flag in NVS), and clear it when a pairing
  completes (security: "Open a trust window for a recorded act, not for missing
  state").
- Refuse pairing through the security callback (`onSecurityRequest()` returns
  false). That fails as "pairing not supported", which keeps the existing bond.

## The Arduino-ESP32 BLE library stores a written value before `onWrite`, and always accepts the write

**Symptom:** After an invalid value is written to a readable and writable
characteristic, a read returns the written value even though the device refused
it and stored nothing.

**Why:** In 3.3.8, `BLECharacteristic` calls `setValue()` in the Bluetooth task,
then `onWrite`, then always answers success (read in the source). `onWrite`
cannot return an ATT error.

**How to apply:**
- Report a write's outcome (accepted or refused) through another characteristic,
  such as a status notification, not through the ATT response.
- For a readable and writable characteristic, have the main loop set the value
  back to the stored content after each write, before reporting the outcome.

## Writes without response from macOS to an ESP32 (Bluedroid) are lost at lengths that leave a 1-byte last fragment

**Symptom:** Sending data from CoreBluetooth on macOS 27 to an ESP32 held as a
BLE keyboard (Arduino-ESP32 3.3.8, Bluedroid) with writes without response,
writes of certain lengths never reached the device's `onWrite`, every time. The
Mac's bluetoothd logged them as sent. No error appeared anywhere.

**Why:** Sending every length from 14 to 252 and from 488 to 500 once, only 84,
174, 245 and 496 bytes never arrived; the content did not matter (filler bytes
were used). Each is a length whose L2CAP packet (length + 7) splits into 90- or
251-byte fragments with a 1-byte last fragment (91 = 90 + 1, 181 = 180 + 1,
252 = 251 + 1, 503 = 502 + 1). Our reading is that the path fragments at both
90 and 251 bytes somewhere and a 1-byte last fragment is lost; which side drops
it was not established.

**How to apply:**
- Keep writes to 244 bytes (one 251-byte packet with the ATT and L2CAP headers).
- That still meets the 90-byte fragments, so avoid lengths where (length + 7)
  mod 90 or mod 251 is 1: send one byte less and carry the rest over.
- Fragmentation may differ between Macs and links. When writes stall, remember
  the length of the first write the device never consumed and avoid it from then
  on (have the device report how many bytes it has consumed).
- It looks like "it stalls sometimes", so recognising a length rule takes time:
  log the length of the stalled write, and confirm by sending each length once.

## When a BLE device notifies its status, hold back on a full transmit queue, and keep going for a while after receiving

**Symptom:** With an ESP32 notifying its status every 15 ms while receiving,
macOS dropped the link every 1.5–2.5 minutes (reason 0x13 on the device). Once
that was fixed, transfers fell to about 0.2 KB/s.

**Why:**
- The device kept notifying while it merely waited mid-operation. Its transmit
  queue filled (`esp_ble_get_cur_sendable_packets_num()` at 0,
  `ESP_GATTS_CONGEST_EVT`, failed sends) until it could not send the response to
  a read from the Mac; macOS dropped the link exactly 30 s after that read (the
  ATT transaction timeout). With a request waiting for its response, the Mac also
  held back the writes after it.
- Notifying only on change let the device sleep between writes (macOS sets a
  slave latency of 22). A 512-byte write spans several packets, so the first
  write after a pause took about 1.1 s to reach a sleeping device.

**How to apply:**
- Send no notification while few transmit buffers are free (fewer than 3): always
  leave room for responses.
- Notify every 15 ms while receiving and for 2 s after the last bytes, to keep
  the device listening; after that, only on change. A screen updated every
  second then keeps it awake.
- On the host, compose the next update only once the device has consumed the
  last one (wait for nothing outstanding, then send the latest state). Updates
  faster than the link otherwise pile up — the screen fell 8 minutes behind.
- Have the host read the status after 1.5 s without a notification, so a held
  back notification costs nothing.

## A BLE keyboard's PnP ID: macOS takes a new one on reconnection and keys the keyboard type by it

**Symptom:** On an ESP32 BLE keyboard (Arduino-ESP32 3.3.8's bundled BLE
library), the PnP ID was changed from a placeholder to real values. Flashed with
the bond kept and reconnected, macOS 27.0 used the new vendor and product IDs
without pairing again, took the device for a new keyboard and opened the
Keyboard Setup Assistant.

**Why:**
- macOS read the PnP ID (Device Information, 0x2A50) again on reconnection, not
  only at pairing: System Information and the HID device in the IORegistry showed
  the new values right after it (one observation, after a reflash).
- The keyboard type (ANSI, ISO, JIS) is recorded per product ID, vendor ID and
  HID country code in `/Library/Preferences/com.apple.keyboardtype.plist` (keys
  `product-vendor-country`, in decimal). A new combination is asked about again.
- The bundled `BLEHIDDevice::pnp()` packs the IDs big-endian; macOS reads them
  little-endian, so passing the IDs as they are swaps their bytes. The
  `pnp(0x02, 0xe502, 0xa111, 0x0210)` many ESP32 keyboards use is pre-swapped
  and, read back, puts Espressif's Bluetooth company identifier 0x02E5 under the
  USB-IF source. Devices carrying the same value share one keyboard-type setting.

**How to apply:**
- Write the seven PnP ID bytes to the characteristic directly, in the order the
  Mac reads them (little-endian), instead of calling `pnp()`. Pin the bytes in a
  test and check that swapping them makes it fail.
- Without a company identifier of your own, use source 0x01 (Bluetooth SIG) with
  the chip maker's identifier (Espressif: 0x02E5; check the SIG's list) and a
  product ID of your own. pid.codes requires a device with a USB interface, so it
  does not fit a Bluetooth-only one; test IDs (pid.codes 1209/0001 and the like)
  are not for devices others use.
- Changing the PnP ID makes the user choose the keyboard type again. A device
  with three buttons cannot press the keys the assistant asks for: tell users to
  skip that step and choose the type.
- The new value was seen taken once, for the PnP ID, on macOS 27.0. The report map and
  the GATT layout may still be kept from pairing: treat changing them as needing
  a new pairing.

## The M5Stack BASIC's backlight does not dim at 44.1 kHz PWM — use 1 kHz, 14 bits and a 2.2 power curve

**Symptom:** Lowering the brightness with M5Unified/M5GFX's
`M5.Display.setBrightness()` on a BASIC v2.7, 1 % (2 of 255) was still bright
enough to read and 50 % was quite bright: there was no dark end.

**Why:** On this board M5GFX drives the backlight pin (GPIO32) at 44.1 kHz with
9 bits and no offset (`_set_pwm_backlight(GPIO_NUM_32, 7, 44100)`). Moved to
1 kHz and 14 bits, the same channel reached a dark end. Pulse width is not the
explanation (1 % is a 0.18 µs pulse at 44.1 kHz; one count at 1 kHz is 0.06 µs,
and that looked dim). A likely cause, not measured: each pulse lights the
backlight a little longer than its width, and at 44 times the pulse rate that
extra adds up.

**How to apply:**
- Right after `M5.begin()`, move the channel M5GFX attached with
  `ledcChangeFrequency(32, 1000, 14)` (Arduino-ESP32 3.x public API: no second
  channel, no change to M5GFX). Write it with `ledcWrite(32, duty)` from then on,
  and never call `M5.Display.setBrightness()` again: it writes 9-bit duties.
- Use a duty of `(percent/100)^2.2` of full (16383), rounded (1 % is one
  count). On the device (2026-09-25, one unit, one person's judgement)
  1 % was barely visible, the steps looked about even, and there was no flicker
  and no sound.
- A curve alone at 44.1 kHz does not help: below 1 % there is one step left.
  Suspect the frequency first.
- Check the board (`M5.getBoard()`) and `ledcChangeFrequency`'s return value,
  and fall back to M5GFX's linear mapping when either fails.
