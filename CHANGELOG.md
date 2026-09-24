# Changelog

## 2026-09-25

- **embedded** (ja + en): macOS 27.0 took a BLE keyboard's new PnP ID on
  reconnection and keys the keyboard type by product, vendor and country code;
  write the PnP ID bytes directly (`BLEHIDDevice::pnp()` packs big-endian), and
  which vendor namespace fits a Bluetooth-only device.

## 2026-09-24 (late night)

- **embedded** (ja + en): writes without response from macOS 27 to an ESP32
  (Bluedroid) are lost at lengths that leave a 1-byte last L2CAP fragment (84,
  174, 245, 496 measured) — cap at 244, avoid those, learn others from stalls.
- **embedded** (ja + en): a status-notifying BLE device must hold back on a full
  transmit queue (or macOS drops the link after a 30 s ATT timeout) and keep
  notifying for a while after receiving (or it sleeps between writes); the host
  composes the next update only when the last is consumed.
- **embedded** (ja + en): the CCCD entry now records that macOS sets up the
  keyboard with the HID CCCDs requiring encryption.
- **macos-gui** (ja + en): a CGBitmapContext's memory starts with the top row.

## 2026-09-24 (night)

- **embedded** (ja + en): the Arduino-ESP32 BLE library's `notify()` sends to
  every connected peer without checking encryption — send private reports to the
  bonded, encrypted connection by id.
- **embedded** (ja + en): the library stores every CCCD write in NVS from any
  peer; make CCCD writes require encryption, and erase stored subscriptions with
  the bond.
- **embedded** (ja + en): Bluedroid deletes the bond of a peer whose pairing or
  encryption fails (from `libbt.a`'s disassembly) — never open pairing because a
  bond is missing.
- **embedded** (ja + en): the library stores a written value before `onWrite` and
  always accepts writes — report outcomes elsewhere and restore the value.
- **embedded** (ja + en): the GATT-layout entry now covers handles, a separate
  layout version, and storing that version even when no bond existed.
- **security** (ja + en): open a trust window for a recorded act, not for missing
  state.

## 2026-09-24 (evening)

- **embedded** (ja + en): a BLE HID keyboard on macOS must make its HID reads
  require encryption, or macOS's HID host gives up while the passkey is typed and
  never retries.
- **embedded** (ja + en): macOS sets an HID device's link to 15 ms / latency 22 /
  data length 90 and reverts requests; writes to the peripheral reach ~4 KB/s only
  while it notifies every connection interval. Keep the host's write queue short.
- **embedded** (ja + en): a CoreBluetooth app can use a custom GATT service of a
  device the system holds as a keyboard (`retrieveConnectedPeripherals` by the
  custom UUID; 0x1812 is hidden).
- **embedded** (ja + en): a bonded Mac keeps the GATT layout; Bluedroid refused to
  send Service Changed (0x82) — fix the layout per release.
- **embedded** (ja + en): with Arduino-ESP32 and BLE only, release the Classic
  controller memory before `BLEDevice::init()` (+21 KB of heap).
- **embedded** (ja + en): arduino-cli build properties — quote each `-D` flag
  separately.
- **shell-scripting** (ja + en): zsh's builtin `log` shadows `/usr/bin/log`; an app
  started with `open` runs from `/`, so pass absolute paths.

## 2026-09-24

- **embedded** (ja + en): with the esp32:esp32 Arduino core, `arduino-cli
  compile --output-dir` also copies the binaries, map, sdkconfig and
  `build.options.json` — with the build machine's absolute paths — into
  `<sketch>/build/<fqbn>/` inside the source tree, through the core's `savehex`
  hooks. Build with `--build-path` under `dist/` instead, and fail the build if
  that folder appears.

- **embedded** (ja + en): arduino-cli treats a quote as special only at the start
  of an argument. Pass a string macro through `--build-property` with the whole
  flag single-quoted (`'-DFW_VERSION="v1"'`) and confirm it in the binary.

- **embedded** (ja + en): the M5Stack BASIC v2.7's CH9102F lost sync at 921600
  and 460800 baud on macOS; 230400 worked for a 4 MB read and a write. State the
  upload speed instead of taking the board's 1500000 default, and copy the flash
  range in use before overwriting a device.

## 2026-09-23

- **shell-scripting** (ja + en): tell "the input could not be fetched" apart
  from an empty answer. `$(cmd 2>/dev/null || true)` folds a failed lookup into
  a legitimate empty result, a silenced fetch leaves a stale ref, and
  `gh repo list --limit` truncates without a word; a health check passed on all
  three. Judge the input once where it is fetched, report it as NOT checked,
  and verify by injecting the fault into the real target.

- **shell-scripting** (ja + en): `git rev-parse` without `--verify` prints a
  missing ref's name to stdout and exits 128, so a `|| echo fallback` branch
  never runs and an assignment ends the script under `set -e`. Read refs with
  `--verify --quiet`.

- **shell-scripting** (ja + en): GNU xargs runs the command once with no
  argument on empty input; macOS xargs does not and drops empty `-0` items.
  With `git -C ""` that runs in the caller's directory. Guard in the called
  code and test it directly, not through the local xargs.

- **development-process** (ja + en): an umbrella's `git fetch` also fetches
  submodules whose pointer moved (`on-demand`), so fetching an umbrella and its
  submodules in parallel writes one repository from two processes. Pass
  `--no-recurse-submodules`; the test needs `protocol.file.allow=always` or it
  cannot fail.

- **development-process** (ja + en): "slow because sequential" — measure by
  section first. Two thirds of a 2:39 run were per-item network round trips;
  one listing call replaced most of them, the rest were parallelized, and the
  run took 0:53.

- **build-and-packaging** (ja + en): correction — check a Linux tarball's
  xattrs in its pax headers (Python's `tarfile`), not by grepping the
  decompressed stream. The grep also matches file text, and a bundled
  CHANGELOG that named the keywords got a clean archive refused.

- **testing** (ja + en): a check that judges by a marker must look for it only
  inside what the marker describes. A whole-file grep for the closed release
  gate's `exit $$rc` passed an open gate because another target of the same
  Makefile carried that string. Take a control sample with the marker outside
  the target too.

- **shell-scripting** (ja + en): `! grep -q` reads grep's error status 2 as
  "no match". A pattern starting with `-` is parsed as an option. Pass
  patterns with `-e` and require status 1 explicitly.

- **build-and-packaging** (ja + en): correction — `COPYFILE_DISABLE=1` does not
  keep macOS extended attributes out of a Linux tarball; bsdtar still writes
  them as pax headers, which the entry listing does not show. Use
  `tar --no-xattrs` and gate on the archive's actual contents.

## 2026-09-22

- **build-and-packaging** (ja + en): prevent AppleDouble entries in Linux
  release tarballs built on macOS and verify the exact archive contents.

- **security** (ja + en): live Slack HTML/JSON round trips exposed text/plain
  metadata and force-download responses; verify attachment identity and full
  size instead of trusting MIME or a sniffed prefix alone.

- **config-and-io** (ja + en): preserve exclusive bounds when converting to
  coarser timestamp precision; floor start and ceil end only when needed.
- **security** (ja + en): error-shaped JSON can be an actual attachment; compare
  metadata and size, and pair refusal tests with legitimate file cases.

- **testing** (ja + en): disable Go's successful test cache when rechecking
  changed execution restrictions; cached skips are not a measurement of the new environment.

- **security** (ja + en): the identity entry, extended from building
  `nlink-jp/pathguard` — anchor a place that does not exist yet by identity too
  (its parent's spellings are unbounded), fold names by Unicode as APFS does,
  stat every ancestor (`/.vol/<dev>` does not stat, `/.vol/<dev>/<ino>` does),
  cap every form a link hop produces, and never let an empty refusal mean
  "allowed". The earlier advice to rely on the name comparison for a missing
  place was wrong and is replaced.
- **security** (ja + en): compare places by identity, not by name — APFS is
  case-insensitive, and four reviews of chrome-pilot-mcp's confinement found the
  same class each time (case variants, a planted temporary name, a swapped work
  directory, a dangling link climbing with `..`).
- **config-and-io** (ja + en): record a guessed value marked apart from a fact,
  and compare only facts (image-forge's architecture check).
- **testing** (ja + en): kitty's graphics protocol takes PNG only and `q=2` hides
  the refusal; when a dependency's cancel waits for the other side, keep the slot
  until it answers, and measure whether it does (spice-vdagent does not).

## 2026-09-21

- **llm-integration** (ja + en): scoped the three-stage claim. Stages one and two
  are properties of an artifact and belong to development — a formula pointing at
  an older release means the thing is not installable at the version it claims.
  Stage three is a property of a machine: on a development machine a resident
  agent holding the binary it started with is the normal state, often the wanted
  one, and unavoidable where the host cannot reload a server. So it is a
  debugging lead for "why is this answering something I already fixed", not a
  release completion criterion and not a gate — the attempt to put it in the
  release checklist was withdrawn for that reason. The measurement itself and the
  field-over-word marker stand as written.

- **llm-integration** (ja + en): two ways a model licence shipped wrong, and the
  guard that catches both. One catalog read the terms off the conversion repo it
  downloads from; another read them off the base model the weights were trained
  from and therefore told users commercial use was permitted, while the weights
  were their publisher's non-commercial licence. Adds that a publisher's family
  is not uniform (large-v3-turbo is MIT while its siblings are Apache-2.0; one
  repo can hold a non-commercial checkpoint beside re-hosted Apache-2.0 parts),
  that both catalogs' default model happened to be the correct entry so
  spot-checks passed, that the provenance belongs in a field pinned by a test
  against the entry count, that a correction has to be shown to reach an
  already-installed model, and that two genuine statements mean reporting the
  stricter and naming both.

- **release-engineering** (ja + en): the marker gate does not cover the recipe that
  follows it. In repos whose `verify-release` predates the template's, the last
  block chained unzip, the packaged binary's `--version` and `spctl` and ended the
  single statement in `|| true`, so a zip that did not unpack passed — found in
  two repos while releasing them. The entry now says to judge each step on its
  own, to require the packaged binary's `--version` to contain the tag (a zip from
  another tag clears both notarization gates and is caught only there), and to
  prove it by control, since check-org compares the vendored scripts and not the
  Makefile recipe.

- **development-process** (ja + en): a pair-parity check does not resolve references. Comparing
  eleven moved-and-translated records to their counterparts reported eleven matches, while the
  links *into* those pairs stayed broken — and a sweep across the organization found 55 dead
  links in 6 repositories, four of them in changelogs, where the link is the only route from a
  release note to the record. Says to resolve each link from the linking document's own
  directory, to report "N of M resolved", to assert that each exemption (code spans, external
  schemes, anchors, vendored copies) is silent, to fix a label that spells a path along with its
  target, and to wire the check into the organization gate the same day.
- **security** (ja + en): a credential may follow a redirect only within the domain the request
  started in. Go re-sends Authorization to the same host or a subdomain but not to a sibling, and
  the target answers an unauthenticated request with an HTML sign-in page at status 200 — so the
  re-attachment is necessary and the linter finding against it reads as a false positive; the
  defect is that it was re-attached to whatever host the redirect named, for ten hops. Covers
  judging against the originally requested URL, approximating a registrable domain so that it
  errs narrow, failing by name rather than withholding silently, justifying the finding in place
  with the tests that cover both halves, and testing a cross-host redirect without DNS.

- **macos-gui** (ja + en): a menu bar panel's own state is the only state you can read at click
  time. `NSPopover.isShown` lags a close by half a second, `popoverDidClose` arrives after a
  show that followed it, and the window's `isVisible` is false while a queued show waits — so
  a re-click read as "the panel is open" closed it again and users saw nothing happen (0 of 10
  on three shipped apps). One click also yields two events, either of which can be missing, so
  they must not be paired by order. Includes the synthetic-click harness that reproduced it,
  and which host to choose: a panel with a pop-up menu inside it has to be a non-activating
  NSPanel, while the rapid re-click residual is the same on either host (measured).
- **macos-gui** (corrected twice, ja + en): the entry added earlier today about
  `CFPreferencesSynchronize` was designed against a failure nobody had measured, and its first
  correction trusted a measurement of a throwaway domain. Measured on macOS 27.0: an unsaved
  write still returns true and the API hides it for up to a minute; the plist is truthful but,
  for the real global ByHost domain, is written 4–8 s after a *successful* change. So at write
  time neither says whether a preference was saved. The entry now says what a read-back can
  and cannot detect, what needs no such knowledge, and to measure a failure on the real target
  before designing for it. The summary of that entry in the item below is superseded.
- Eleven entries from an audit of one week's changes across the organisation and the repairs
  that followed (ja + en). **config-and-io**: a terminal query read from a goroutine leaves a
  reader behind that takes the terminal's next input (`/dev/tty` is a blocking descriptor on
  macOS; read with `select(2)` on the calling goroutine, to the last reply); a time budget is
  stated once, or the error names the wrong setting and a longer budget is silently cut; a
  checked number can still become a zero Duration, which means "unset" and "no timeout"; a path
  in hand is not fed back through a grammar that finds paths in text. **macos-gui**: after
  `CFPreferencesSynchronize` fails the process cannot believe its own reads — the record names
  both states and the process stops; two repairs were withdrawn in review first.
  **containers-and-infra**: podman's `name=` filter is an unanchored regex. **security**: a
  sandbox-writable directory is reached through an `os.Root` opened from the directory someone
  vouched for, and the source check is an allow-list. **testing**: a fake that answers every
  question the same way cannot see a wrong question; a relaunch is a new object; a refusal
  test needs a positive control. **development-process**: a lesson nobody is held to is a
  note; line citations are remapped by diff from the base copy; an independent pass converges
  in rounds, and the author's fix is where the next defect is.

## 2026-09-20

- **mcp-server-design**: four entries from wrapping an unauthenticated public API. An upstream
  that silently ignores (an invalid `sort`, paging, an unknown vendor, an unknown parameter, a
  parent ID that does not exist, a `vendor` that does not filter — six of one kind): validate
  every value you send, do not declare an argument upstream ignores, and pin the traps live.
  Upstream's "no data" arrives as a positive (zero hosts with a zero time, empty lists, an
  unfiltered list): convert it, flag an unverifiable empty answer as `incomplete`, and treat
  the positive side the same way. `not_found` is decided on the 404's body, not the status — a
  missing route answered 401, and a listing has no single resource to be absent. And a
  response budget is not closed by naming fields: behind the per-part caps, shorten and name
  any other long string, and withhold a result over the byte ceiling whole. The skeleton entry
  gains "check which sibling is newest, package by package".
- **security**: Go's `url.Error` quotes the whole URL, so a config validation error can leak a
  credential — report the inner `Err` only. And text for a terminal is made inert once over the
  whole result, not at each print site (five sites had been forgotten); JSON output gets the
  characters `encoding/json` leaves raw written as the `\u` escapes they equal; the test fills
  the wire types by reflection.
- **testing**: checking guards by mutating a copy, and the three kinds of "not caught" (a hole
  in the guard, a test that never takes the reader's path, a redundant mechanism hiding the
  absence); a release gate that skips is a gate that passes; drift checks against primary
  sources are written with tolerances, on one shared pacer; and the write path that turns a
  typed escape into the raw character — three times — closed by a test over every source file.
- **config-and-io**: a key that is present is a value that was given (`""`, NaN and a flag
  sentinel were each read as "not set"), and cache keys are hashes over length-prefixed parts,
  not joins with a separator the parts may contain.
- **llm-integration**: a shared space between agent sessions stays empty without independent
  observers and a trigger to write. A machine-local board worked end to end and was archived
  after sixteen days: 386 sessions registered, 12 ever posted, no record was corroborated or
  disputed. Reading was enforced and writing optional; the quality model assumed observers that
  one operator's sessions never provide; what was posted already lived in git, memory and
  project documents. Count the independent-observer pairs and the marginal information before
  building, ship the keep-or-kill counters with the feature, and treat push delivery as a cost
  paid every turn.
- **macos-gui**: two lessons from rehosting a menu-bar popover in a non-activating `NSPanel`.
  Nothing tells such a panel that the user went elsewhere: mouse-down monitors cover one way
  of leaving, and Cmd-Tab and a Space change need `NSWorkspace`'s notifications on the same
  close path — found by the pre-release review, not by a report; list what the popover did
  implicitly before removing it. And right after `NSStatusItem.length` changes, the item's
  window has the new width at the old origin; the menu bar corrects it 29–41 ms later with a
  `didMoveNotification`. Follow that notification — waiting one run loop turn left the panel
  57 pt off, and a longer wait would be the same guess.
- **macos-gui (correction)**: the entry added earlier today that told a popover with controls
  to activate the app on open was wrong, and is rewritten. Right after launch macOS refuses the
  activation request, so the first click activated the app after all and ended the menu 71 ms
  in — found only with the signed bundle launched through LaunchServices; the nine-of-nine check
  behind the original advice used a build started from a terminal. The remedy is a
  `.nonactivatingPanel` `NSPanel` that never asks for activation (menu open for 1,906 ms under
  the same condition). Also recorded: a hidden title bar still gives SwiftUI content a 32 pt
  safe area (`safeAreaRegions = []`), `NSApp.isActive` reads true while such a panel is key,
  and the lookup that would have prevented this is by the mechanism's name, not the symptom's.
- **testing (correction)**: `strings` cannot confirm that a diagnostic recorder is absent from
  a Swift binary — literals of 15 bytes or fewer are stored inline and appear nowhere; the
  diagnostic build also gave 0. Count symbols with `nm` instead, and see a check for absence
  come up positive once before trusting its negative.
- **macos-gui**: seven lessons from building a menu bar network meter, each measured on
  macOS 27.0. A popover with controls has to activate the app when it opens — otherwise the
  first click's activation ends the menu that click just opened, 78 ms in. A panel refreshed
  every second must hold refreshes while one of its menus is tracking, or the binding's
  setter receives the old value. With the default animation `popover.isShown` stays true for
  534–546 ms after a close is requested, which loses every other open click at two clicks a
  second. A bar in a small scrolling graph is a sample, not a time bucket, and motion has to
  be looked at as a filmstrip. Things that must look aligned are placed from one source and
  the offset measured in ink. A status item's button reports the menu bar's own appearance,
  not the system's. A field is sized for the longest real value, not for the sample data.
- **testing**: the script reading an event trace declared a working fix broken twice in one
  day — the recorder's monitor runs after the app's, and a binding's setter runs before the
  menu's end notification. When the person who used the app and the script disagree, read
  the raw lines first.
- **config-and-io**: per-interface byte counters read through `sysctl` sit in 64-bit fields,
  but an unprivileged process on macOS 27 is handed the true value modulo 2^32, floored to
  1 KiB — found only by bracketing the reading with the bundled `netstat`, which gets the
  unaltered value. A field's width does not guarantee the value's width. Take deltas modulo
  2^32 always (flooring and the modulus commute), discard any sample whose interval was
  stretched by sleep, and do not read "they matched" on a fresh machine as "no truncation".
- **testing**: a menu bar status item cannot be observed through the window list on
  macOS 27 — the app owns no window for it and no process gains one. An empty result is not
  evidence that nothing is shown; print the total window count so that "cannot see" is not
  mistaken for "does not exist".
- **macos-gui**: a menu-bar `NSPopover`'s `.transient` dismissal closed it only for outside
  clicks that landed in a window taking activation; clicks on an empty stretch of the menu
  bar or on another process's non-activating panel were missed every time (macOS 27.0). The
  defect had been handed over as "`makeKey()` activates the app and activation breaks
  `.transient`" — a control build without `makeKey()` behaved identically and the app never
  became frontmost, so the `makeKey()` entry's activation claim is now marked as an
  inference. Install global + local mouse-down monitors unconditionally, build a control
  with the suspected cause removed before fixing, and never judge dismissal by a click on a
  normal window alone.
- **macos-gui**: the same measurement repeated on the second menu-bar app, the one the
  "activation breaks `.transient`" reading came from. A control build without the monitors
  gave identical numbers whether the app had never been activated, had its settings window
  open, or had opened and closed it — activation history changed nothing. With `makeKey()`
  removed as well, nothing closed at all, where the first app's `makeKey()`-less control had
  behaved like the original: a control build's result does not carry over to another app.
  Also recorded: a frontmost accessory app stops being frontmost when its status item click
  opens the popover.
- **macos-gui**: on macOS 27 a click on an app's own status item reaches its global
  mouse-down monitor before the button's action (the menu bar is hosted by another process),
  so "monitor closes, action toggles on `isShown`" turns a re-click into close-then-reopen —
  visibly so with `animates = false`, and masked only by the close animation's margin
  otherwise. The two events cannot be matched by identity (the action runs under a
  synthesized mouse-up with event number 0) and, while the app is active, the action
  sometimes never arrives. Match them by order instead, take the note only for clicks inside
  the status item button's *window* frame read at click time with top-left ownership (the
  screen's top row is exactly `frame.maxY`), and always put a re-click — inactive, active,
  and at the item's edges — among the verification cells.
- **macos-gui**: `NSEvent.removeMonitor` over-releases a monitor removed twice. Syncing the
  monitor from `popoverDidClose` crashed on quit from a button inside the panel, because
  `applicationWillTerminate` had removed it and kept the reference. One removal site that
  drops the reference, a test that counts the sites, and the app's exits (judged by exit
  status and crash reports) among the verification cells.
- **development-process**: a formatter target with no pinned configuration applies the tool's
  defaults to the whole tree — a scaffolded, never-run `make fmt` rewrote all 39 files of a
  4-space Swift repository to 2-space. Configuration could not rescue it: across 18 Swift
  repositories, fixing only the indentation still changed 451 of 583 files, because the
  pretty-printer re-lays-out line breaks. Land a formatter with its configuration and a
  no-diff run, measure the diff before adding one to existing code, and size the remedy to the
  measured scope (one hook in the whole organization: delete it, add a scaffold-checklist item).

## 2026-09-19

- **macos-gui**: a control bar was laid out correctly, not hidden, not transparent,
  and invisible — a content layer inside a neighbouring view measured 1024x752
  where its view was 1024x676, and the parent did not clip. View frames can never
  show this; dump the layer tree. Clip a dependency's drawing view to your own
  frame as a structural guarantee, and note that a reproduction harness only
  establishes what is not the cause.
- **testing**: a gate proved an injected key reached the guest kernel and passed for
  its whole life while the guest's X server ran with zero input devices, so no
  application in the guest ever saw a keystroke. Name the layer each test observes,
  add a check for the gap between it and the layer a user experiences, break a
  never-failing check on purpose once, and have a person look at drawing and input
  paths at least once. Added after moving one such observation up a layer: the
  observer has to be running before the event it observes, which means before the
  readiness marker the harness waits on, not after it.

## 2026-09-18

- **shell-scripting**: `$?` read after a whole `if` statement is the statement's
  own status, which is 0 when the condition is false and there is no `else`. A
  retry helper classified failures correctly and still returned success for the
  pass-through path. Read the condition's status inside the branch; `cmd &&
  return 0` is not an alternative under `set -e`.
- **containers-and-infra**: publishing an ephemeral port allocates it and binds
  it a moment later, with nothing reserving it in between, so a container start
  can lose the race to whatever released a port just then (once in three gate
  runs, right after the previous phase stopped). Retry the start with a fresh
  allocation, gated on that one error text, and say so on stderr.
- **testing**: a transfer that stalls at the same round number every run points
  at a credit window, not at the byte handling — a 16,000-byte chunk needed
  eight tokens against a ten-token grant replenished five at a time. Also: size
  stability proves nothing as a completion signal when the receiver preallocates
  the file; key on modification time, always emit a line at a deadline, and take
  the receiver's own digest as the evidence.
- **development-process**: hashes over a vendored tree prove nobody edited it
  unrecorded, not that the recorded patches still explain it — replay them
  against the pinned upstream. And a local patch is the least-reviewed code in
  the repository: when a dependency misbehaves, read your own diff against it
  before reading its implementation.

## 2026-09-17

- **config-and-io**: Bubble Tea's `Program.Send` from inside Update freezes
  the whole UI, Ctrl+C included, and the callbacks that run inside Update —
  slash handlers, settings application, hook execution — do not look like
  they do. Recorded after the same mechanism did it a second time, with the
  remedy (return the notification, or `go Send` and stop claiming delivery),
  the review heuristic (enumerate every Send caller and ask whether it can
  run from Update), and the regression shape that catches it: a sender that
  never returns.

## 2026-09-17

- **macos-gui**: an app's appearance is decided by the SDK recorded in
  `LC_BUILD_VERSION`, and the Xcode 27 / Swift 6.4 `swift build` stamps that
  field with the deployment target instead — a re-released menu-bar app came
  back drawing with the previous generation of window chrome, with an identical
  Info.plist and no source cause. Pass `-platform_version` explicitly (SDKROOT
  and a lone `-sdk_version` do not work), derive the minimum from Package.swift,
  and gate `verify-release` on the linked SDK: build, signature, notarization and
  tests all pass either way, and the symptom only shows when the deployment
  target is older than the SDK, so a sibling app looking fine proves nothing.

## 2026-09-17

- **testing**: an interval cut must fire when a segment *touches* the boundary,
  not only when one straddles it — contiguous segments leave nothing straddling
  an instant that a state change or a gap split lands on exactly, and a limit
  that is only recomputed once cleared turns that single miss into a permanent
  one. The defect disabled both a new day-boundary rule and an already-released
  session cap, and it stayed asymptomatic because firing depends on the phase of
  the sampling tick, and a real multi-week history almost never met it. Tests
  for this class vary step
  and phase in a table; and a wall-clock boundary makes the logical day 23 or 25
  hours across a DST change, so "never exceeds 24h" needs the crossing test
  before it is written down.

## 2026-09-17

- **testing**: verify web-to-native authority through real WebKit/HTTPS callbacks, immutable native approval, and isolated certificate fixtures.

## 2026-09-17

- **testing**: two terminal lessons from inline images in an inline TUI.
  Output that occupies N rows must erase below itself — the inline
  renderer's flush clears one row, so the old frame survives to the right of
  a narrower picture on every row it covers, with the row accounting
  perfectly correct. And a capability probe sends a device-attributes
  request alongside its capability query, because the capability question
  has no negative answer: silence cannot be told from slowness, and the
  measured cost of reading silence as "no" was the full 2.001 s budget at
  every start on Apple Terminal, plus the query's own body printed on the
  screen. Both classes are decidable only on a real terminal.

## 2026-09-15

- **macos-gui**: record spacing-only fixture measurements, human visual confirmation, exact absent-key restoration, and serialized watchdog recovery.

- Extend macOS 27 menu-bar validation lessons with bounded live-trial AX evidence, allowed-control checks, and permission-toggle mismatches.

- **macos-gui**: separate macOS 27 private API discovery, GUI access,
  Accessibility trust, item extraction, actual control, and restoration.
  Record the execution scope and OS build; do not interpret missing access
  as successful hiding, or a runtime signature check as functional proof.

## 2026-09-13

- **testing**: a withdrawn mechanism disappears from the code, not from the
  prose. Two servers shipped with tool descriptions and a `get_usage` error
  table still instructing the model to use the mechanism that had just been
  removed; a third shipped a schema declaring an argument optional that the
  handler required. Prose is not compiled, and in MCP it is part of the product
  — so keep one list of retired terms and walk every string the model reads.
  Also: `go test ./...` never builds tagged suites, so add
  `go vet -tags <tag> ./...` to the gate, and count a withdrawal sweep in
  surfaces (descriptions, usage, manifest, `--help`, every language's README,
  setup guides, config examples) rather than in code.
- **mcp-server-design**: the per-call output-root entry is amended with the
  settled contract — one name (`work_dir`), one meaning (a directory the
  *caller* can read back), resolved argument → `_meta["jp.nlink/work_dir"]` →
  error with no server-owned default behind it. Measure the channels first: of
  four calling runtimes, only the per-call argument reaches all four (`roots`
  is unanswered or answers the project directory, one strips the environment,
  cwd holds only for per-session stdio servers). A server whose product is
  *data* takes no work directory at all — it caps rows and counts the
  omission. Operator allowlists are replaced by a fixed credential blacklist,
  because a prefix list cannot name "the project"; and the blacklist must
  compare the path as given *and* symlink-resolved against each entry as given
  *and* resolved, since a home directory whose `~/.ssh` is itself a symlink
  defeats either spelling alone.
- **security**: a gate asks whether an operation is allowed, so an injection
  that accepts the operation and corrupts only a value passes every layer — the
  sandbox, the credential list and the human gate all engage on *which
  operation, on what target*, and the call they see is correct. Count layers by
  the failure mode each stops; rendering arguments in the approval prompt helps
  but is clipped, and recognising a wrong value needs knowing the right one. The
  control for this class is a diff review, which is operational, not a mechanism.
- **security**: measuring injection resistance with loud attacks overstates it.
  Loud payloads (discard-all-instructions, forged authority, forged closing tag)
  scored 0 in 500 against a capable local model wrapped or not; task-consistent
  payloads that dictate one output field scored 92–100% unwrapped. Wrapping cut
  one of them to 0.3%, another to 14%, and a third not at all — so the
  mechanism's effectiveness is a property of the attack-and-task pair, not of
  the mechanism. Vary the target field, always run the benign twin, report a
  table rather than a rate.
- **llm-integration**: rates need n>=100 and a benign twin. The same experiment
  at n=20 produced the opposite conclusion from n=100; an earlier record's
  headline effect carried the same width and should not be quoted as an effect
  size. Also: a classifier that tests for a substring cannot tell a defended
  answer quoting the payload from an obedient one.
- **macos-gui**: a menu-bar popover IS scriptable — walk the AX tree by hand
  for text and state, click with CGEvent (System Events' `click at` may not
  fire SwiftUI tap gestures), aim at an element's centre, and re-read the tree
  before every click, because one extra banner shifts every row below it.
- **macos-gui**: in a polling UI, the poll's result and the action's answer
  must not share one field. An action that re-polls when it finishes erases
  its own error about 100 ms later, so a refused request reads exactly like a
  click that never happened; split the channels, let only the next action or
  closing the panel clear the action's word, and pin the rule in a pure value
  type — the UI layer cannot catch that regression.
- **security**: before writing "layer X covers this instead", count whether X
  can see the call in the shape it judges — a matcher that judges a path
  argument cannot cover a walk that has none, and the degraded fallback the
  ADR promised was never there. If the fallback cannot cover it, refuse the
  operation while degraded rather than adding a mode branch, which puts the
  safety on the path exercised least.
- **testing**: a design document that says "applied at every N" needs, in the
  same commit, a test that enumerates N. A test pinning a list of names does
  not close a class of call sites; and when fixing, look for the line two below
  that quietly undoes the fix.
- **llm-integration**: a tool description is a fact inside the prompt, not a
  comment — change it in the commit that changes the behaviour, keep
  implementation promises out of it unless a test can fail on them, and grep
  every description for the words whose behaviour just changed.
- **development-process**: a mechanism two sibling products share is not fixed
  when only one of them is fixed. Name the shared mechanisms in BOTH products'
  agent-facing documents, say what the rule is not, and record any deliberate
  divergence.
- **mcp-server-design**: if a tool returns a file, the output root is a per-call
  argument — the only value that works is the caller's per-session directory, so
  a startup flag becomes runtime-specific and an unset `${VAR}` in a shared
  registration expands silently to empty. Reject unknown fields so a misspelled
  root is not ignored, require an absolute path, validate on the call that
  supplied it, and treat "can the caller open the path I returned?" as part of
  the success case.
- **security**: remove your own namespace from children rather than keeping it
  — the environment has no bounded domain on either side (a secret-name
  denylist misses `OPENAI_KEY`; a needs-allowlist is just as open-ended), so
  delete the rule, leave the operator's environment alone, and invert the
  prefix: a runtime's own variables reach no child while its exports do, closed
  by a test over the whole namespace. Grep the org before calling a namespace
  yours. **Supersedes** the earlier "re-evaluate namespace exemptions" entry.
- **security**: counting a protected list's enforcers is the symptom — an entry
  is cheap and a rule is the smell, so where a kernel is available move the
  boundary there instead of growing the matcher. The enumeration tools then
  hide nothing. **Amends** the per-operation entry below, whose
  skip-and-report half is withdrawn.
- **security**: `file-read*` and `file-read-data` are not the same Seatbelt
  operation — the first covers metadata, and a Go `os.Root` listing stats every
  entry, so one denied name silently emptied every walk.
- **testing**: when you add a boundary, drive the shipped artifact through the
  shipped boundary in the same commit — stubs and `chmod 000` pass while
  production is broken — and drive the path that turns the boundary on, which
  is where the next release of the same design failed: the test existed, the
  installer was never run, and the feature shipped inert.
- **security**: a protected-path list is enforced per operation, not per tool
  family — count the read tools among a credential list's enforcers, mirror
  the lanes (an operator-only Review for a single-file read), and pin every
  enforcer with a test.

## 2026-09-12

- **security**: close SSRF inside the HTTP dialer as a finite address domain
  — resolve, judge every address (embedded IPv4 unwrapped), connect to the
  vetted literal as one operation; refuse redirect targets with userinfo; carry
  no credentials; honour the loosening setting only from the default config
  path.
- **mcp-server-design**: page a fetched document by character offset with a
  document id, hold it in memory for the turn, and derive the default page size
  from the consumer's byte cap and the language's byte ratio.

- **config-and-io**: a retention period is not a retention
  depth — ingestion lag eats into the new end, and an expiration set before a
  backfill finishes silently deletes the oldest days as they land.

- **llm-integration**: in a tag-delimited stream with no escaping, honour only
  JSON-bodied tags and pair openers one at a time; check a metered API key
  with a request that cannot succeed, and map errors by the body's code
  before the HTTP status.
- **testing**: a single observation is not a rule — record the counts, word
  reader-facing text after the observation, and correct every surface at once.

- **mcp-server-design**: correct the claim that MCP has no cancellation
  notification — `notifications/cancelled` exists since 2024-11-05; the receiver
  may ignore it, so kill-and-respawn remains the reliable stop.
- **development-process**: "the spec has no X" is unverified without a
  primary-source citation; an implementation's comment is not evidence about
  the spec.

## 2026-09-09

- **security**: re-evaluate namespace-wide environment exemptions when a port
  introduces API credentials; protect child environments and reject unsupported
  credential storage paths with regression tests.

- **llm-integration**: distinguish literal source field values from summary
  verification labels; verify actual edits and post-summary answers separately,
  with exact semantic checks and explicit JSON envelope handling.

- 9 new entries from a two-report review of an agent runtime (a whole-project
  review and a review of its auto-approve mode), their fixes, and the design
  decisions that followed:
  - **security**: a constraint derived from conversation may only ever
    tighten — one that can loosen is a permission, and the proposer cannot
    be its own judge; state a judge the operator's *intent*, never the
    control, or the model reports "not permitted" about something nothing
    prevents and the user reads a falsehood; the only history safe to feed a
    judge is what the operator typed, and provenance — not message role —
    is what selects it
  - **mcp-server-design**: a synchronous write to a child's stdin is
    unbounded, so a deadline created after it supervises nothing and a lock
    held across it turns one wedged peer into a total stall; discarding the
    ok of a type assertion on a required argument turns absence into a
    destructive zero value, and two parse paths for one tool means only one
    of them carries the check
  - **config-and-io**: replace-by-rename drops the file mode, and leaving the
    mode to the call site is how one of two call sites gets it wrong
  - **shell-scripting**: `|| true` at the end of an `&&` chain forgives the
    whole chain, not the last command
  - **testing**: a gate that only ever runs by hand, on valid input, needs a
    self-test with synthetic bad artifacts in the routine check
  - **development-process**: an index check that matches by identifier walks
    past a rename — a number is not a link

## 2026-09-08 (3)

- 1 new entry from an operator report about a TUI's scrollback and the
  pre-release review of its fix:
  - **development-process**: a change that funnels call sites into one
    helper is enumerated by behaviour, not by grep — grep returns only
    the places that already have the code, and the place that needs
    fixing is the one that does not; the miss lands on the early return
    beside the success path in the same block. Funnelling is not proof
    of coverage: lower a universal claim until it is true.

## 2026-09-08 (2)

- 1 new entry from a post-release field report against the CLI agent's
  settings panel and the pre-release review of its fix:
  - **development-process**: reuse a "reload" path only when the
    granularity of the change matches the granularity of the reload — a
    one-row toggle that called the whole-set reconnect respawned 25
    processes inside the UI's event loop; and an edge documented as
    "bounded, acceptable" for transcript attribution (name-prefix
    matching) became a live-tool loss the moment the same primitive
    drove removal — re-review such allowances in the commit that changes
    their caller.

## 2026-09-08 (1)

- 3 new entries from a CLI agent release that added name-based MCP tool
  exclusion, cut the startup banner to what nothing else will say, and ran six
  independent verification passes before tagging:
  - **testing**: an independent pass finds its real defects right after a fix —
    each fix creates a new, unverified surface, so the brief asks what the
    previous fix brought in; assertions in ADRs/READMEs are grepped against the
    source before commit (names only — meaning is the reviewer's charge); the
    tree is not touched while a pass runs; convergence is judged by the trend
    of severity, and the last pass is narrowed to the last commit.
  - **development-process**: a report is not a control — "print it at startup"
    as the remedy for a hazard is an indulgence; a startup line needs to be a
    change, name the next action, and rarely appear; close the banner by a type
    of facts and cut "write it and it appears" wiring.
  - **llm-integration**: a tool the model must not use is made absent by name,
    not refused — MCP has no capability field and its annotations are untrusted;
    declarations measured at 92% of the prompt; one exact-name predicate serves
    declaration and dispatch, with no patterns, profiles, or deny value.

## 2026-09-06 (3)

- 1 new entry from a bulk documentation edit that quietly folded submodule
  bumps into five umbrella commits:
  - **development-process**: `git add -u` in an umbrella silently sweeps up
    submodule pointers — a gitlink is a tracked file, and `git status --short`
    shows it as one `M <tool>` line indistinguishable from a documentation
    edit. Name the paths, read `git diff --cached --stat` before committing,
    and keep pointer bumps in their own commit; splitting is only practical
    before the push.

## 2026-09-06 (2)

- 2 new entries from separating third-party licence notices across the
  organization, and from the broken submodules a bulk migration ran into:
  - **release-engineering**: appending to LICENSE breaks GitHub's licence
    classification, and splitting the notice out must ship with the artifact —
    the attribution's home is the distribution, not the repository, so the fix
    for the classification silently removes it from releases unless the
    packaging changes in the same commit. Two repositories had never shipped
    the notice for code they bundle, including the one consulted as the
    precedent for the split.
  - **development-process**: an embedded `.git` inside a submodule only bites
    when you remove it — everyday commit/push/status work normally, and
    `deinit`/`rm` aborts midway through a bulk migration. Detect it by asking
    whether `<submodule>/.git` is a directory; repair with
    `git submodule absorbgitdirs`, which produces no commit.

## 2026-09-06 (1)

- 2 new entries from building a container-based Linux test harness for the
  Go repositories that cross-compile a linux/* binary, and from the archived
  repositories that the harness's work list swept up:
  - **testing**: a cross-platform test harness detects its own defects first
    — a locally cached amd64 image runs the whole suite under qemu behind a
    single warning line (surfacing as `signal 11`, which reads as a real
    crash), container root bypasses DAC so a test expecting a 0500 write to
    fail sees it succeed, and `GOTOOLCHAIN=local` turns a newer go.mod into
    "fails on Linux". Run both platforms and diff them; repeat a difference
    and measure its flake rate before calling it platform-specific.
  - **development-process**: archived state lives only on GitHub, so a work
    list built from local files grabs read-only repositories — one of them
    held a real defect that could not be fixed. Separate archived projects
    into their own umbrella keeping the originating series as a directory
    level, and make an archived repo inside an active umbrella a failing
    check rather than a skipped one.

## 2026-09-05 (6)

- 1 new entry from the review response of a macOS ZIP tool (ADR-0005
  there):
  - **security**: clean up from what was created, not from what was
    planned — an ownership ledger fed only by exclusive creates (files
    `O_EXCL`, directories `mkdir(2)`), `lstat`-based uniqueness, claims
    before the first write, one scratch arena per operation owned by the
    output's owner, aggregate memory budgets over per-unit thresholds, and
    source-reading tests that pin the class.

## 2026-09-05 (5)

- 1 amendment after reports of an unofficial port of a CLI agent runtime
  (rebuilt for WSL without the sandbox):
  - **security** (amended "Bound an agent's shell with the kernel's cage"):
    verify confinement itself, not only the unasked lane — a stubbed
    sandbox check passes as confined and runs everything at the
    model-approvable tier; probe the write lane at startup and degrade to
    "every command asks the human" when a probe fails.

## 2026-09-05 (4)

- 1 amendment from an external finding on a CLI agent runtime's
  protected-file rules:
  - **security** (amended "Pin an agent's trust to content"): name checks
    fold case on macOS — the default APFS volume is case-insensitive, so a
    protected-name rule comparing exact bytes protects nothing; fold in the
    shared rule, canonicalize name→key mappings, and test with variants
    plus a real-sandbox run that includes controls.

## 2026-09-05 (3)

- 1 entry and 1 amendment from a CLI agent runtime whose whole-system
  documents drifted under a "docs in the same commit" rule, and whose
  banner notes cited ADR numbers again on real-device use:
  - **development-process** (1): the rule names no document — keep a
    change-kind → documents routing table in the agent briefing file,
    treat a structural `fix:` like a `feat:`, and make the routable rows
    mechanical (every internal package, options callback and subcommand
    must be documented); symmetry checks cannot see this class.
  - **development-process** (amended "Status output is not
    documentation"): design references never reach the screen — an AST
    test over operator- and model-facing packages fails on any string
    literal citing an ADR number; and before a release, collect every
    operator-facing string into one document and have a reader who did
    not write the change go through it with the rubric.

## 2026-09-05 (2)

- 2 entries from a CLI agent runtime that moved project trust from the
  directory name to the content it consumes, and from the E2E run that
  overwrote the repository's own briefing file on the way:
  - **security** (1): pin an agent's trust to content — digest through the
    same root the loader reads (link targets included), mark an empty set
    as recorded, trust-on-first-use only interactively, re-pin only the one
    approved write and only when the pin was current as it began (a
    before/after hook pair), shell commands report instead of re-pinning,
    decide trust before any project read, one grant object every loader
    takes (fixed by an AST test), pin edits under the policy lock.
  - **testing** (1): an E2E script aimed at a fixture must validate its
    target directory against a pattern (`set -u`, a `case` guard) — an
    unquoted heredoc expanded its argument to "" and `cd ""` ran the script
    in the repository; pin the required sections of agent briefing files
    in `make check`.

## 2026-09-05

- 1 entry from a usage-aggregation CLI whose session listing lost its
  time order when the producer switched session ids from timestamps to
  UUIDs (parsing was unaffected, so every test stayed green):
  - **config-and-io** (1): an id-ordered listing is chronological only
    while ids are timestamps — carry first / last record time on the
    aggregated row and sort by it; write the keys always (no
    `omitempty`) so absence means only an older producer; refuse a time
    sort on a dense series whose filler rows have no time rather than
    adding a comparator fallback.

## 2026-09-04 (2)

- 1 entry from a CLI agent runtime whose one-shot startup was silent
  for 5-7 s in a few runs out of a dozen:
  - **llm-integration** (1): `cloud.google.com/go/logging`'s
    `client.Logger` auto-detects the monitored resource by fetching
    from the GCE metadata server unless `CommonResource` is set; on a
    Mac the link-local fetch blocks on the kernel's ARP probe and its
    2 s dial timeout is retried as transient, so the cost depends on
    the neighbour cache. Declare the `global` resource the detection
    falls back to, pin it with a hit-counting `GCE_METADATA_HOST`
    fake, remove an unintended wait rather than announcing it, and
    catch an intermittent startup mode with an env-gated per-step
    trace over a dozen runs.

## 2026-09-04 (1)

- 1 correction from re-measuring a Vertex endpoint claim after a model
  generation went GA:
  - **llm-integration** (1): the Gemini 3 family is served from `global`
    **and the `us` / `eu` multi-regions**, not `global` only — the
    2026-08 measurement had seen the `us-central1` 404 and never tried a
    multi-region. Entry retitled; the client-side 404 hint now names all
    three; new rule of thumb: record a negative conclusion together with
    the candidates actually tried — an untried one is "unverified", not
    "does not work".

## 2026-09-02 (4)

- 2 entries from a usage-accounting CLI + menu-bar app that showed a new
  default model's turns as $0 for the second time:
  - **llm-integration** (1): when syncing a price table, check every
    multiplier column and the footnotes, not just the base price — a
    footnoted 0.025× cache-read rate moved the total by ~2× on a
    workload where cache reads are 69% of cost; keep multipliers
    per-model, never write future price schedules into comments, and
    let config override per-model multipliers as a no-release stopgap.
  - **macos-gui** (1): a CLI's stderr warning never reaches its GUI —
    put deliberate "$0 / skipped" states into the JSON contract, derive
    them from stored rows (they survive restarts and updates), keep the
    fields optional on the GUI side, and put the exit (a Reprice
    action) in the same box as the state.

## 2026-09-02 (3)

- 1 entry from an org health check that summarized all-green after
  checking nothing (wrong caller cwd + fail-open skips):
  - **shell-scripting** (1): an operational script must not infer its
    root from `$PWD` — derive it from `BASH_SOURCE` and keep the
    explicit argument as override; and a checking script must count
    skipped targets into the verdict (INCOMPLETE, non-zero exit)
    instead of warning and summarizing green. Either defect alone is
    survivable; composed they manufacture a green run out of zero
    checks.

## 2026-09-02 (2)

- 3 entries from retiring a chat-diagram tool whose feedback loop was
  measured firing once in 76 sessions, while its fence prohibition had
  bred hand-drawn box art:
  - **llm-integration** (2): "do not do X" over-generalizes — a
    prohibition blocks the exit but leaves the demand, which the model
    routes through an unanticipated, unguarded third path; remove the
    trigger instead of prohibiting, and when the wanted behavior
    matches the model's trained prior, say nothing and pin the silence
    by test. Plus a sequel to "rejecting without telling the author":
    the value of an in-turn feedback loop is a claim to measure, not
    assume — a loop almost never exercised does not justify a standing
    per-use cost.
  - **testing** (1): before calling something "measured", ask whether
    the probe input represents the real thing — a space-free probe
    line concluded a renderer "never wraps code-block lines" while
    real box art (which contains spaces) was word-wrapped and sheared;
    build probes and expectations from real artifacts, and record a
    measured claim's input provenance.

## 2026-09-02 (1)

- 1 entry from reviving a dormant delegated-search tool (zero
  spontaneous firings in 75 sessions):
  - **llm-integration** (1): model-facing guidance has layers — system
    prompt beats tool description, and in-band text inside tool
    results beats both; put a feature's trigger in the strongest
    layer, audit every layer for counter-triggers naming the
    competing path, scope in-band caveats to the action that needs
    them, and verify by transcript both that the feature fires and
    that the follow-on behaviour changed.

## 2026-08-31 (3)

- 1 entry from a TUI feature's pre-release testing (deny-with-reason
  dialog field):
  - **config-and-io** (1): call Focus() on a bubbles component before
    storing it into the Bubble Tea model — a pointer-receiver call
    after the value copy mutates only the local variable, and an
    unfocused textinput silently drops every key; fix the order
    (create → configure → Focus → assign), keep the returned tea.Cmd,
    and pin it with a KeyMsg-after-store regression test.

## 2026-08-31 (2)

- 2 entries from the same day's failure retrospective (lint restoration,
  release round, ADR-0058 risk-ladder gap):
  - **development-process** (2): absence of findings is not success — a
    panicking linter, a fixer whose "0 findings" meant a broken build, and a
    release step that skipped silently are the same failure, so pair every
    zero with proof the checker ran and audit the whole result surface after a
    mechanical sweep; and changing a boundary invariant means dispositioning
    every consumer of the old one — grep the identifier that encodes the
    boundary before shipping, record each consumer's disposition in the ADR,
    and keep audit wording in step with the boundary, because the bug class is
    "component nobody re-asked", not "wrong code".

## 2026-08-31

- 2 entries, and one correction, from taking a fleet of seven file-mediated MCP
  servers off `workspace_root` and giving the client the guard instead
  (abuse/asn/mac/malware/otx/rdns/urlscan-lookup, gem-agent):
  - **mcp-server-design** (1 + 1 correction): a server cannot know the model's
    context window, so bounding a response is the caller's job and spilling an
    oversized one is the client's — seven servers had each grown the same
    threshold, and the file escape hatch made every one of them depend on the
    client owning a filesystem it could *name*. Reachability, not the file, is
    what has to survive the removal. This supersedes the older advice to offer
    an untrimmed result as a file: a trim should be escapable in place, with a
    knob that raises the cut.
  - **development-process** (1): a linked worktree breaks in any repository that
    needs `core.worktree` — enabling `extensions.worktreeConfig` without
    migrating that setting leaves every linked worktree resolving its work tree
    onto the git directory itself. The condition is what `git config --get
    core.worktree` returns, not whether the repository is a submodule.

## 2026-08-30 (5)

- 3 entries from connecting a stdio-to-HTTP MCP bridge to a provider with no
  dynamic client registration (mcp-bridge, GitHub setup guide):
  - **mcp-server-design** (3): RFC 9728 metadata coming back does not mean DCR
    is available — the two discovery stages are independent, so check for
    `registration_endpoint` with one curl instead of inferring it from a failed
    login (and mind that RFC 8414 inserts the well-known segment before a
    path-bearing issuer); a tool list is context paid for every session, so
    servers past ~20 tools should let clients pick a subset at connect time by
    individual name, because group-level selection barely moves the number; and
    enumerate the capability surface before building a control layer, because
    what the API permits is not what the server exposes — a permanent limit
    belongs in the credential's authority, not a denylist of tool names.

## 2026-08-30 (4)

- 2 entries from replacing an LLM-authored-JSON transcription tool with one
  built on a specialised model (gem-scribe, ADR-0001):
  - **llm-integration** (2): if a specialised model exists, stop stacking
    mitigations on a general LLM writing your structure — the mitigations
    tolerate a freehand document rather than making it correct, and a salvage
    pass buys survival by silently discarding content; and once the structure
    is guaranteed, the remaining errors turn quiet, so the failure shapes
    validation cannot catch need names and one result field to carry them.

## 2026-08-30 (3)

- 2 entries from migrating an image-generation CLI to the current model
  generation (gem-image, ADR-009):
  - **llm-integration** (2): the Gemini 3 family is served from the global
    endpoint only, so a generation migration has to move the model name and
    the location together — plus a free `:countTokens` availability probe and
    a client-side hint for the uninformative 404; and "this model always
    returns PNG" does not survive a generation change, because the
    lightweight image tier returns JPEG where flash and pro return PNG.

## 2026-08-30 (2)

- 1 entry from making a CLI agent's sessions priceable (gem-agent,
  ADR-0057):
  - **llm-integration** (1): the API reports tokens and never money, so
    usage must be written down at call time — one accounting record per
    call with its source and model, bucket semantics measured
    (thoughts bill as output, cached is a share of prompt), and exactly
    one place that counts.

## 2026-08-30

- 1 entry from a false stall warning on a CLI agent (gem-agent,
  ADR-0056):
  - **llm-integration** (1): a function call arrives as one whole
    part, so nothing — not a chunk, not a byte — reaches the client
    while the model composes a large argument (measured 40s of dead
    wire for a 21KB write); set stall thresholds from that
    measurement, and keep the supplier's reason off the screen.

## 2026-08-29 (2)

- 1 entry from piped-stdin support on a CLI agent (gem-agent,
  ADR-0055):
  - **security** (1): keep piped stdin out of an LLM agent's trusted
    instruction channel — carry operator-chosen-but-not-written data
    on the same nonce-wrapped lane as tool results, and pin the
    boundary with a test on the trusted-side payload.

## 2026-08-29

- 2 entries from headless approval-control work on a CLI agent
  (gem-agent, ADR-0053/0054):
  - **security** (1): with no human present, degrade "ask the human"
    to "deny with the reason" instead of killing the ladder; arm
    unattended automation per-invocation on the command line, never
    from a standing config file.
  - **testing** (1): a constant that bounds a feature needs a reach
    measurement — a 3-round window set by intuition covered only 30%
    of real evaluations, and every beyond-window escalation was
    hand-approved friction.

## 2026-08-28

- 1 entry from a field report on a CLI agent's navigation degrading in
  grown projects (gem-agent, ADR-0052):
  - **llm-integration** (1): enumeration tools break at scale unless
    ignore-aware — 99.3% of the walk was generated content and it was
    the noise; two-layer skipping (builtin list + gitignore semantics),
    filter enumeration only, report every skip, cross-check a
    hand-written gitignore matcher against git check-ignore, and
    distribute caps so nothing starves.

## 2026-08-27

- 1 entry from a field report on a CLI agent destroying large documents
  it was asked to revise (gem-agent v0.50.0, ADR-0051):
  - **llm-integration** (1): an agent revising a large document destroys
    it by summarizing — the harness manufactures the failure (economy
    steering, whole-file writes, mid-task compaction, truncation caps);
    floors: a declared-intent shrink guard, a regeneration rule, a
    staleness warning in the compaction stand-in, and a size delta on
    the approval UI.

## 2026-08-27

- 1 entry from an operator's UX sweep of a CLI agent's command output
  (gem-agent v0.49.1–v0.49.2):
  - **development-process** (1): status output is not documentation —
    sort operator-facing text by the question it answers (session
    facts in output, feature explanation in docs, teaching in empty
    states, per-event disclosures stay); help is a map, exits get a
    receipt, and never hand-wrap sentences in source strings.


## 2026-08-26 (5)

- 1 entry from rebuilding a withdrawn approval learner as a risk
  rulebook the LLM judge reads (gem-agent ADR-0050):
  - **security** (1): let the record advise the judge, never write the
    policy — guidance degrades judgment while rules open bypasses;
    layer it (hand-written base + reviewed learned text), frame
    blanket-approval prose as escalation evidence, keep the channel
    independent of the proposer's, and live-verify both directions.


## 2026-08-26 (4)

- 1 entry from withdrawing an approval-rule learner the operator judged
  dangerous after real use (gem-agent ADR-0049):
  - **security** (1): a human confirmation step is not a durable
    boundary for granting standing permissions — primed one-keystroke
    consent, bundled risk, momentary evidence buying permanent grants,
    and post-consent invisibility compound; prefer observability over
    grant automation, enumerate or make grants ephemeral, and build
    the management surface before the granter.


## 2026-08-26 (3)

- 1 entry from an approval-rule learner that proposed nothing on its
  first real session (gem-agent ADR-0048):
  - **testing** (1): a feature that learns from usage must be tested
    against usage-shaped data — fixtures written to satisfy a
    threshold cannot falsify it; ask "is this bar reachable?"
    separately, reproduce before diagnosing, and check whether the
    friction even has the shape you are counting.


## 2026-08-26 (2)

- 1 entry from building approval-rule learning from an operator's
  recorded decisions (gem-agent ADR-0045):
  - **security** (1): count sessions, not calls — a session allowlist
    turns one keystroke into many approvals, so a call-counted
    threshold measures how often the agent asked; plus the syntactic
    shared key, recording the key with the decision, keeping the
    learner model-free, and proposing rather than applying.

## 2026-08-26 (2)

- 2 entries from an agent that asked for approval of a `cp` without ever
  saying why (gem-agent ADR-0047):
  - **llm-integration** (1): a thinking model's tool-call preamble goes
    to thoughts, not text (measured: 1 text part in 349 tool-calling
    turns) — give intent a required tool argument instead of inferring
    it or asking for prose, inject it centrally, strip it before the
    call runs, and surface its absence rather than refusing.
  - **security** (1): a model-authored "why" is for the human only —
    strip it before any LLM evaluator reads the call, or the evaluator
    is handed the proposer's own justification as evidence.

## 2026-08-26

- 1 entry from teaching an auto-approve evaluator the semantics of MCP
  tools (gem-agent ADR-0046):
  - **security** (1): server-authored metadata can be evidence for an
    LLM judge — as a claim, never a fact: the metadata author equals
    the effect author, so it is never a safety mechanism but adds no
    new trust either; frame contradiction and self-argument as
    escalation evidence, and live-measure both directions.

## 2026-08-25 (2)

- 1 entry from an un-notarised zip shipping with green checks when
  Apple's updated developer agreement broke the notary probe (2026-08):
  - **release-engineering** (1): a fail-open step plus a verifier that
    only displays equals a defective release with green checks — gate
    on a local success marker, keep gated commands out of pipes, grep
    for the success token instead of tailing output, and test the gate
    in both directions.

## 2026-08-25 (1)

- 1 entry from porting the org's Claude Code pre-tool guard into the
  fallback CLI agent (2026-08):
  - **security** (1): guards that live outside the agent must travel to
    the fallback, and their contract is measured from the real artifact
    — the installed guard denied via stdout JSON, not the documented
    exit code, and never read the tool name at all.

## 2026-08-24 (2)

- 1 entry from moving diagram rendering behind a tool in a fallback CLI
  agent (2026-08):
  - **llm-integration** (1): rejecting is honest, but rejecting without
    telling the author is not — route the rejection and its reason back
    to the model, preferably by making the verification a tool call, and
    make every reason actionable.

## 2026-08-24 (1)

- 4 entries from a false hardware-failure alarm on an internally run
  DNS server, where a daily log-summary report attributed
  two-year-old kernel errors to "yesterday" (2026-08):
  - **testing** (1): establish a log's timestamp semantics before
    drawing any conclusion about time — year presence, timezone
    (writer, viewer, mixtures, DST), event versus ingestion time,
    monotonicity, relative-time conversion. Includes recovering the
    year from a year-less log via the line numbers of date-string
    matches.
  - **containers-and-infra** (3): rsyslog ships its logrotate config in
    a separate `rsyslog-logrotate` subpackage, so without it only the
    five files rsyslog writes grow unbounded while everything else
    rotates normally; passing an individual config file to
    `logrotate -f` discards `/etc/logrotate.conf` globals and silently
    falls back to `rotate 0`; and `PerSourcePenalties` (OpenSSH 9.8+,
    default on in 9.9) makes an SSH liveness check lock out the checker
    itself, producing a symptom indistinguishable from a storage I/O
    hang.

## 2026-08-22 (14)

- 1 entry from a documentation audit of a fallback CLI agent that found
  43 discrepancies across seven releases (2026-08):
  - **development-process** (1): an en/ja mirror check that verifies
    only pairing does not protect content — compare the identifiers a
    translation must not change, include the root READMEs, and measure
    the false-positive rate before adopting the rule.

## 2026-08-22 (13)

- 2 entries from a cross-session memory feature that had never fired in
  a fallback CLI agent (2026-08):
  - **llm-integration** (1): a model-facing capability written as "you
    can" never fires — state the trigger, balance the positive against
    the prohibitions, and count proposals rather than trusting
    precision (0 proposals in 39 sessions looked like a precise
    feature).
  - **security** (1): self-approval is not a defence — never let the
    party that proposed an action approve it; exclude persistent,
    irreversible, or privilege-escalating operations from
    auto-approval, and re-measure the approval path once a dormant
    feature starts firing.

## 2026-08-22 (12)

- 1 entry generalizing the day's diagram-rendering lessons in a
  fallback CLI agent (2026-08):
  - **llm-integration** (1): silently correcting dynamic LLM output is
    a bad move — the only valid responses are teach, verify+reject, or
    surface to the human; the sole exception is meaning-preserving
    parsing. Includes why it is structurally bad (unbounded shifting
    input, the cheapest lever ignored, inverted failure mode,
    source/display divergence).

## 2026-08-22 (11)

- 1 entry from an operator preferring instruction over correction in a
  fallback CLI agent (2026-08):
  - **llm-integration** (1): teach the accepted dialect in the system
    prompt before writing a rewriter — measured compliance, avoids
    meaning-changing corrections, and fixes the model's own output;
    keep existing translations as a frozen backstop after measuring
    what removing them costs.

## 2026-08-22 (10)

- 1 entry from an operator calling out accumulated special cases in a
  fallback CLI agent (2026-08):
  - **llm-integration** (1): stop bolting per-construct blacklists onto
    a renderer — fold the design into translate / fit / verify and let
    the generic verification be the single gate; one such blacklist was
    written from an unverified assumption and refused correct output.

## 2026-08-22 (9)

- 1 entry from a verification guard disabling a feature in a fallback
  CLI agent (2026-08):
  - **llm-integration** (1): a fidelity guard can kill the feature by
    false negative — strip the renderer's own decoration from both
    sides before comparing, and test guards against real rendered
    output rather than hand-written art (the bug hid because
    single-word labels never tripped it).

## 2026-08-22 (8)

- 1 correction from operator feedback on a fallback CLI agent's
  diagram rendering (2026-08):
  - **llm-integration** (1, corrected): guard derived renderings
    against being wrong, never against being ugly — a readability
    threshold (the previous entry's advice) was reverted by the
    operator: show what fits, and let the human tell the model "too
    complex". The correction loop lives in the conversation.

## 2026-08-22 (7)

- 1 amendment from dense diagrams breaking in a fallback CLI agent
  (2026-08):
  - **llm-integration** (2, amended): layout quality is a limit
    fidelity checks cannot phrase — cap readable complexity
    (relationships, per-node degree) independently of width; recurring
    breakage means you have hit the renderer's expressiveness limit.

## 2026-08-22 (6)

- 1 amendment from a diagram drawn wrong in a fallback CLI agent
  (2026-08):
  - **llm-integration** (1, amended): a fidelity guard must count
    structure (edges vs arrowheads), not only label presence — a
    mis-parsed edge-label syntax produced a plausible wrong graph with
    every label present.

## 2026-08-22 (5)

- 1 entry from fixing sheared box art in a fallback CLI agent's TUI
  under a Japanese locale (2026-08):
  - **config-and-io** (1): one width model per Go TUI — go-runewidth
    flips East Asian Ambiguous glyphs to two cells under a CJK locale
    and glamour pads code blocks with it; pin EastAsianWidth=false
    (honour an explicit RUNEWIDTH_EASTASIAN), and test new
    width-measuring dependencies with the wide setting forced.

## 2026-08-22 (4)

- 1 entry from adding terminal rendering of mermaid diagrams to a
  fallback CLI agent (2026-08):
  - **llm-integration** (1): when a derived rendering replaces source,
    decide the renderer by measuring your own inputs, keep the
    advertised capability and the implementation as one tested list,
    verify fidelity (every source label present) before substituting,
    and fall back to source on any loss.

## 2026-08-22 (3)

- 1 entry from a whole-code review of a fallback CLI agent (2026-08):
  - **security** (1): a permission justified by "only a human writes
    this input" becomes a hole the moment delegation lets a model write
    that input — audit input-channel trust premises when adding
    sub-agents, grep the comments for the premise, and test
    containment at the input preprocessing layer.

## 2026-08-22 (2)

- 1 entry from rebuilding a fallback CLI agent's round limit
  (2026-08):
  - **llm-integration** (1): make an agent's round limit an
    intervention ladder, not a guillotine — deterministic loop
    detector that escalates early, a model progress review at the
    threshold, per-mode decisions, an absolute cap no verdict lifts,
    and stop messages that teach recovery instead of destroying it.

## 2026-08-22

- 1 entry from adding instruction context to a fallback CLI agent's
  auto-approval (2026-08):
  - **security** (1): give LLM auto-approval the operator's typed
    instruction as alignment evidence — the one context an injection
    attacker cannot write; wrap it as evidence, bound it structurally
    to early rounds, and live-probe the reach (Safe-tier calls never
    see the evaluator).

## 2026-08-21 (6)

- 1 entry from adding a delegated file-search sub-agent to a fallback
  CLI agent (2026-08):
  - **llm-integration** (1): narrow child-agent delegation to
    read-only single-purpose — approval forwarded from an invisible
    context is not approval; loud-failing positive allowlists, no
    recursion by construction, deny-all approver as fail-closed
    insurance, labeled audit events, live delegation display.

## 2026-08-21 (5)

- 1 entry from adding workplace audit logging to a fallback CLI agent
  (2026-08):
  - **security** (1): default agent audit telemetry to the cloud the
    tool already authenticates to; send metadata only (never
    prompts/contents); keep telemetry config global-only so a cloned
    project cannot plant an exfiltration sink; and never let telemetry
    failures hurt the tool.

## 2026-08-21 (4)

- 1 entry from a cancellation deadlock in a fallback CLI agent's shell
  tool (2026-08):
  - **config-and-io** (1): exec.CommandContext kills only the direct
    child — a grandchild holding the inherited pipe blocks Wait forever,
    defeating the timeout first and the interrupt second; always pair
    Setpgid + group SIGKILL + WaitDelay, and give the UI an escape
    ladder for tools that ignore cancellation anyway.

## 2026-08-21 (3)

- 2 entries from a second whole-code review of a fallback CLI agent
  (2026-08):
  - **testing** (1): an injected feature's weakest point is its single
    production call site — unit tests inject the dependency themselves
    and a nil-default options field degrades gracefully enough to hide
    the missing wire; pin constructor literals with a go/ast test, and
    E2E every surface the feature passes through.
  - **config-and-io** (1): cloud-storage writers commit buffered data on
    Close — abort by cancelling the writer's context; in a
    content-addressed permanent store, also hash and upload from one fd
    and re-hash the stream with a verifying reader.

## 2026-08-21 (2)

- 2 entries from an operator question about parallel session-id
  collisions in a fallback CLI agent (2026-08):
  - **config-and-io** (1): os.WriteFile creates the file empty before
    writing — marker/flag files read by other processes must land by
    temp+rename, with an empty file treated as unowned and repaired.
  - **testing** (1): verify concurrency concerns with a concurrent test
    rather than by reading the code — the test written to prove the
    suspected layer safe caught a ~50%-frequency race in the adjacent
    layer.

## 2026-08-21

- 1 entry from adding a self-information tool to a fallback CLI agent
  (2026-08):
  - **llm-integration** (1): hand the agent its own runtime (model name,
    context occupancy, limits) through one read-only tool, rendered from
    the same accounting struct the human UI reads; select fields by
    whether they change model behavior, and register before the agent
    constructor if it caches tool declarations.

## 2026-08-20

- 3 entries from extending a fallback CLI agent (memory, GCS media, UI
  language; 2026-08):
  - **security** (1): agent-writable memory is a persistence vector — an
    injected instruction that survives into every future session; gate the
    write, not the read.
  - **llm-integration** (1): a stale ADC `quota_project_id` 404s every GCS
    call while Vertex keeps working (Vertex carries the project in the URL
    path); pin the intended project via `GOOGLE_CLOUD_QUOTA_PROJECT`.
  - **config-and-io** (1): fix mixed-language UI with one message struct and
    two complete per-language catalogs, enforced by a reflection
    completeness test (plus fmt-verb agreement); resolve POSIX-style once at
    startup and declare the surfaces that stay English.

## 2026-08-19 (2)

- 1 entry from writing and then running a monthly drill runbook for a
  fallback CLI agent (2026-08):
  - **testing** (1): a runbook is unfinished until it has been run once —
    its first run rewrote three of its seven steps, each of which read
    correctly and verified nothing. Three shapes to suspect: a question
    answerable without traversing the path under test, a check that depends
    on the subject's cooperation, and a check whose result varies between
    runs. Pin state-dependent defaults at the top, keep the verdict binary,
    and include one real task done with the tool alone.

## 2026-08-19

- 5 entries from adding session resume and context compaction to a CLI coding
  agent (2026-08):
  - **llm-integration** (4): Gemini thought signatures measurably replay across
    processes, which makes verbatim replay a working basis for session resume
    under the same model — and makes refusing a different model the honest
    design, since stripping signatures is a 400. Persisting conversation history
    for resume belongs in one log, not a log plus a parallel transcript, which
    forces that log's conversation records to be lossless; clipping them for
    readability produces a resumed session that has forgotten half of a file it
    read, with nothing to announce the gap. A history-compaction summariser is
    summarising untrusted data: no tools, nonce-isolated input, the summary
    quoted as data on the way back in, and every failure path leaving the history
    untouched.
  - **testing** (1): a feature can pass every unit test and never fire once in
    production — unit tests supply the gating condition directly, so they never
    check that it arises. Swing the threshold to an extreme in a real run and
    watch it fire; and apply the invariant to the case the feature exists for,
    because a safety-motivated rule can exclude it without looking wrong.

## 2026-08-18

- 1 entry from adding percent and a burn-rate projection to a menu-bar budget
  display (2026-08):
  - **testing** (1): unit tests over pure functions cannot see whether a string
    fits its real width, and driving the live app with synthetic events is too
    leaky to use for every layout change. A SwiftPM executable target is
    importable from its test target, so the view can be laid out in an
    `NSHostingView` at the production width and cached to a PNG — same layout
    engine, same fonts, nothing on the user's screen. Render one image per
    branch, because what breaks is the branch with the longest string.

## 2026-08-17

- 3 entries from building a menu-bar NVMe health monitor (2026-08):
  - **macos-gui** (3): `isTemplate` is honoured for `NSStatusItem.button.image`
    and ignored for an image embedded in an attributed string, so a symbol meant
    to follow the menu bar's colour was drawn in the app's `labelColor` and read
    as grey. An SF Symbol name that does not exist returns nil and degrades the
    status item to a fallback glyph in silence, so every name a renderer can emit
    is asserted to resolve — which needs a test target for the executable, not
    just the core. And sizes measured against the content of the day broke the
    layout three times as sections were added: declare floors and ideals, define
    a shared constant when two places must agree, and stop reserving space for
    text the caller already shows.

## 2026-08-16

- 2 entries from diagnosing whether SMART data can be read from a USB-attached
  external SSD on macOS (2026-08):
  - **testing** (2): an auto-enumeration mode (`--scan`) listed neither drive
    under investigation and returned one unrelated empty drive caddy instead,
    whose "no media" error was then reported as evidence for a constraint it
    said nothing about — the conclusion was independently correct, which is
    exactly why the faulty evidence never surfaced. And: a position in the OS
    device tree was used to infer which physical port a drive occupied, and was
    wrong, because USB traffic from a Thunderbolt port surfaces under the same
    SoC USB controller. Covers reconciling enumerator output by independent
    identifier, calibrating layout against a device of known location, and
    reading one plane's "no device connected" as signal rather than absence.

## 2026-08-15

- 1 entry from a menu-bar status watcher whose panel started looking dark on
  macOS 26 (2026-08):
  - **macos-gui** (1): a status item click does not activate an accessory app,
    so an `NSPopover` that is only shown never becomes key and macOS draws its
    material in the inactive state. Liquid Glass made that inactive rendering
    read as a dark, dimmed sheet, so unchanged code was observed as a
    regression. `makeKey()` right after `show(relativeTo:)` fixes it and is
    pixel-identical to activating the app. Includes the measurement method —
    and why a window-only `screencapture -l` cannot judge a translucent panel.

## 2026-08-10

- 2 entries from building a threat-intel lookup CLI + MCP server against a live API (2026-08):
  - **mcp-server-design** (1): a `limit` that caps the record list does not bound
    the response. Aggregates and reference lists are computed over the whole
    result set, so they grow with the input's popularity — `limit: 3` still
    produced 162 KB and the client refused it. Budget the whole response, trim
    the ranked tail, and account for every value dropped.
  - **testing** (1): a lookup that folds a *failed* source into an empty result
    set reports "not found" for "could not ask". Caught only in a live run — one
    endpoint returned 429 while another returned zero, and the tool printed a
    clean verdict and exited 0. Mock tests cannot reach it, because the failure
    exists only when one source fails and another succeeds. A negative answer is
    the one nobody double-checks.

## 2026-08-09

- 1 entry from finding three repositories whose READMEs called shipped tools
  unreleased (2026-08):
  - **release-engineering** (1): a status written at scaffold time is **not
    updated by releasing**. One README had shipped four times, carried a Homebrew
    formula, and printed that `brew install` two lines under its own
    "not released yet" banner. State status through the presence of install
    instructions, not in prose; if you must write it, put it only where the
    release procedure already goes. Mechanically detectable.

- 1 entry, plus a correction to yesterday's, from following up the escape hatch
  that yesterday's entry recommended (local transcription tool, 2026-08):
  - **llm-integration** (1): whisper.cpp's **grammar-constrained decoding is not
    a vocabulary hint**. Its penalty only subtracts from tokens the grammar
    *rejects* — nothing lifts what it allows — so a permissive grammar carrying
    the wanted name gives output byte-identical to no grammar at all. A grammar
    tight enough to bite collapses the transcript and still never emits the name.
    Fix proper nouns after transcription, and record that you did.
  - **Correction**: the initial-prompt entry pointed at `grammar_rules` as "the
    mechanism" for constraining vocabulary. That was written from the API surface
    rather than measurement — the same failure the entry itself warns about — and
    is now corrected in both languages.
- 1 entry, plus a counter-example on an existing one, from evaluating a
  preprocessing step that an already-linked runtime exposed for free (local
  transcription tool, 2026-08):
  - **development-process** (1): an unused feature of an already-linked runtime
    looks free. Check the **model licence before the technical evaluation** —
    done the other way round you discard something you have proven works — and
    measure **per target**, because "it helped" hides "it did nothing for the
    thing we wanted".
  - **llm-integration**: the model-licence entry warned against classifying too
    conservatively. Added the case pointing the other way — two models, MIT code,
    **nothing at all said about the weights**. Undeclared is not a licence.

- 1 entry from a free-space budget check that refused every extraction onto a
  file server (ZIP utility, 2026-08):
  - **config-and-io** (1): `volumeAvailableCapacityForImportantUsage` answers
    only for local APFS volumes and returns **0, not nil**, on network mounts
    (SMB/NFS) — a space check that takes it at face value reads "0 KB free"
    against a server with terabytes available. Accept the key only when
    positive, fall back to the statfs-backed `volumeAvailableCapacity`, and
    keep the refusal for genuinely full disks (statfs reports ~0 there too).

## 2026-08-08

- 1 entry from a tactics document whose escalation ladder had been outgrown by
  the fleet it ranks:
  - **development-process** (1): a ranking document must have its **endpoints**
    re-derived whenever the group it ranks grows, not just its membership. An
    MCP tactics book ranked servers by how observable a query is and named
    "the target sees a visit from urlscan.io" the ceiling; a browser
    automation server shipped afterwards contacts the target from our own IP,
    strictly above it. Every existing row was still correct — the false part
    was the top, which reads as "nothing is worse than this", and the omitted
    rung is the one taken without deliberation. Two corollaries: scope such a
    document by **capability, not purpose** (the browser server belongs in an
    OSINT book because it *can* touch the target), and prefer "most X do Y;
    exceptions are A, B, C" to "every X does Y" — the same review found a
    blanket "every server ships `get_usage`" against three that ship none.

- 3 entries from a menu-bar app whose panel knew its state and never said it:
  - **macos-gui** (2): a framework error type's full case table has to be
    measured before any error UI is designed — `TranslationError` returns
    `"Unable to Translate"` for seven of its eight cases and bridges every one
    to `NSError` code 1, so a shipped error line rendered an unsupported
    language pair, an internal fault and a missing model identically, under a
    hardcoded guess that was wrong in most of them; and a state in which the
    app *deliberately* does nothing (IME composing, input too short to
    identify, debounce armed) must still be nameable in the UI, which a
    `Bool` cannot do — five such correct decisions had accumulated, all
    silent, and the compound effect reads as a hang.
  - **testing** (1): an LSUIElement menu-bar app can be driven for E2E with
    `CGEvent` + `CGWindowList` + `screencapture -l <windowid>` where
    AppleScript reaches neither the Carbon hotkey nor the status item — with
    the caveat that missed synthetic keystrokes land in the user's frontmost
    application, so it photographs states reliably and drives text input
    unreliably.

## 2026-08-03

- 1 entry from nine ADRs that had to move out of an organization log:
  - **development-process** (1): which log an ADR belongs in is decided by
    what the record binds — and the question has to be forced at writing time
    by a mandatory `Binds` header field, because format and placement are
    learned by imitating existing records, an unstated alternative never
    beats a documented default, and prose criteria mis-sorted hybrid
    retire-and-design records twice.

- 2 entries from measuring a platform constraint instead of working around it:
  - **macos-gui** (1): a notification posted with `trigger: nil` is presented
    by the posting process and withdrawn when it exits — which is why apps
    linger after finishing. One scheduled with a
    `UNTimeIntervalNotificationTrigger` belongs to `notificationd` and
    outlives the process: measured, the banner was still on screen at t=5.0 s
    with the app gone since t=0.57 s. Removes the wind-down that the earlier
    deferred-`terminate` entry exists to make safe.
  - **macos-gui** (1): a Finder multi-selection arrives as one open event, and
    macOS replaces each banner with the next from the same app, so reporting
    per item leaves only the last one readable. One request means one progress
    bar, one question, one report, one reveal — plus the rules for partial
    failure and truncated lists.

- 1 entry from a timer that outlived the state it was scheduled for:
  - **macos-gui** (1): a process kept alive after "finishing" (to let a
    notification banner play out) still receives open events, so an
    uncancellable deferred `terminate` kills whatever arrived in the
    meantime — a measured case truncated a 700 MB extraction to 543 MB with
    no error, and killed a password prompt mid-keystroke. Covers the two
    requirements (cancellable handle, re-decision at fire time), why the
    rule belongs in a pure function, and the reverse check that the app
    still quits at all.

- 1 entry from two keyboard shortcuts that were never wired in a shipped app:
  - **macos-gui** (1): standard editing shortcuts (⌘X/⌘C/⌘V/⌘A/⌘Z) and ⌘W
    reach the first responder only through main-menu key equivalents, so a
    hand-built menu without an Edit menu makes ⌘V in a text field do nothing,
    with no code to breakpoint. Includes the reason the mistake survives
    review — the menu bar draws the top-level item's own title, and the app
    and Window menus are special-cased into appearing without one.

- 1 entry from a launch crash that reached two shipped macOS apps:
  - **macos-gui** (1): never use SwiftPM's `Bundle.module` inside an `.app` —
    it looks beside the bundle root, not in `Contents/Resources`, and falls
    back to a compile-time absolute `.build` path, so it resolves only on the
    machine that built it. Includes the release-verification step that
    reproduces a foreign machine.

- 15 entries from building a menu-bar sensor-monitoring app over a metered
  third-party API:
  - **macos-gui** (7): MenuBarExtra pushing a height onto its content; ForEach id
    uniqueness across a whole List; timers needing the `.common` run-loop modes;
    elapsed-time labels needing a TimelineView; not disabling a control on an
    ambiguous status; asking for OS permission at the moment of intent; changes
    in a second ObservableObject not reaching views that do not observe it.
  - **development-process** (2): building the budget into the design when the
    API is metered and exhaustion looks like an auth failure; preventing
    duplicate workers with a conditional no-op rather than a protocol.
  - **testing** (2): green tests proving nothing when the expectation itself is
    wrong; verifying a GUI on the assumption that what you can see and what runs
    are independent.
  - **release-engineering** (2): payload size giving away a silently skipped
    bundling step; pinning the version string when building something to bundle.
  - **config-and-io** (2): searching both config conventions on macOS;
    long-format storage making re-import idempotent.
  - **security** (1): putting the restraint in the client when the credential is
    more powerful than the use.

## 2026-08-02

- Initial compilation (ADR-015): 13 themed documents in Japanese and English,
  compiled from ~100 engineering-knowledge memories accumulated across
  nlink-jp projects.
