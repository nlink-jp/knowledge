# macOS GUI Apps

Traps and established patterns from developing native SwiftUI / AppKit apps and
Wails (Go + WebView) apps. Each entry follows **symptom → why → how to apply**.
For signing and notarization, see [release-engineering.md](release-engineering.md).

---

## SwiftUI / AppKit (mostly menu-bar resident apps)

### Menu-bar apps cannot be built with SwiftUI alone

**Symptom:** Menu-bar resident (`LSUIElement`) apps require AppKit knowledge for
activation policy, panels, and focus management.

**Key points:**
- The `.accessory` policy cannot take keyboard focus → switch to `.regular` when
  showing the panel, back to `.accessory` when hiding.
- `NSPanel`'s `isMovableByWindowBackground` conflicts with TextEditor et al. →
  set false and allow dragging only by the title bar.
- SwiftUI `@FocusState` + `onAppear` fires too early on first `NSHostingView`
  creation → delay with `DispatchQueue.main.asyncAfter`.
- Info.plist cannot go into an SPM resource bundle → keep it at the project root
  and copy into the bundle from the Makefile.
- **App Nap freezes background update timers**: window-less LSUIElement apps get
  classified as background; `Timer.scheduledTimer` stops and displayed values go
  stale (correct right after launch, stuck after hours — easy to miss). Fix: hold
  the return value of
  `ProcessInfo.processInfo.beginActivity(options: [.userInitiatedAllowingIdleSystemSleep], reason:)`
  for the app's lifetime (system sleep stays allowed). Also refresh user-visible
  views explicitly in `.onAppear`.
- **LSUIElement cannot focus the `Settings` scene / `SettingsLink`** → open
  settings as a regular `Window` + `openWindow(id:)` + activation.

### Never attach a bottom safeAreaInset to NavigationSplitView

**Symptom:** Adding `.safeAreaInset(edge: .bottom) { footer }` to a
`NavigationSplitView` made the sidebar's bottom bar (+/- buttons) completely
invisible, painted over by the footer — and it shipped that way.

**Why:** A bottom safeAreaInset shrinks the split view's own frame but **not the
sidebar column**. The sidebar keeps drawing at full window height, so its bottom
bar lands in the same band as the footer.

**How to apply:**
- Place the footer as a VStack sibling:
  `VStack(spacing: 0) { NavigationSplitView {...}; Divider(); footer }`.
- For any SwiftUI "something disappeared" symptom, measure geometry first. A
  throwaway harness — `NSHostingView` in a bare `NSWindow`, built with `swiftc`
  in minutes — can compare several layouts numerically (pass the real Views/Model
  sources and swap only the App file for a before/after proof on the real view).
  Capture the actual screen with `screencapture -x -o -R<x,y,w,h>`;
  `NSView.cacheDisplay` doesn't render effect-view internals and in-process AX
  walks don't surface SwiftUI frames.
- These rendering regressions are unit-untestable — record them as traps in the
  project's AGENTS.md.

### Don't run live actions during IME composition (marked text)

**Symptom:** A debounced auto-translate fired mid-kana-conversion, processed the
unconfirmed string, failed language detection, and the OS threw a
language-selection dialog over the panel, completely blocking input.

**How to apply:**
- SwiftUI's `TextEditor` doesn't expose marked text. Wrap `NSTextView` via
  `NSViewRepresentable` and check `hasMarkedText()`.
- `textDidChange` alone is insufficient — **no notification fires when the
  confirmed string equals the marked string**, so confirmation goes undetected.
  Override `setMarkedText` / `unmarkText` / `insertText` to signal composition
  start/end directly.
- Also trigger processing on the composition-end edge (composing true→false).
- Extract the decision rules into pure functions and unit-test them (real IMEs
  can't be automated).
- A "skip auto-run when the input language is undetectable" guard prevents the
  OS dialog at the root. Never gate manual execution.

### NSTextView only draws its caret in a key window

**Symptom:** Opening a panel via global hotkey, macOS refuses to activate the app
(another app stays frontmost); the view is firstResponder yet no caret appears.

**How to apply (close all three paths):**
1. In an `NSPanel` subclass, declare `canBecomeKey = true` explicitly
   (`canBecomeMain` stays false).
2. Call `makeKey()` on show, then **re-assert key and focus one runloop later**
   (`NSApp.activate()` is async and can be refused; `activate(ignoringOtherApps:)`
   is deprecated — use `activate()` on macOS 14+).
3. Observe `NSWindow.didBecomeKeyNotification`; on becoming key, re-take
   firstResponder and call `updateInsertionPointStateAndRestartTimer(true)`.

Verify by screenshot that the caret is visible **while another app remains
frontmost**.

### Two traps of resizable menu-bar NSPanels

**Trap 1: `hidesOnDeactivate = true` makes toggling misfire.** Auto-hide leaves
`window.isVisible == true`, so an `isVisible`-based toggle calls `orderOut` on an
already-hidden panel ("first click does nothing, second click opens").
→ Don't set `hidesOnDeactivate`; to close on click-away, call `orderOut`
yourself in `applicationDidResignActive` (which sets `isVisible` correctly).

**Trap 2: without `.nonactivatingPanel` the panel won't appear for ~30 s after
launch.** A normal NSPanel isn't drawn unless the app is active, and macOS 14+
focus-stealing prevention can refuse activation for up to ~30 s right after
launch — `isVisible == true`, position correct, nothing on screen.
→ Add `.nonactivatingPanel` to the styleMask (draws and accepts keyboard input
without app activation). This is why NSPopover-based menu-bar apps never hit
this.

Related lessons:
- Panels persisting size via `setFrameAutosaveName` should clamp to the current
  screen's `visibleFrame` on show (a size saved on a big display can go
  off-screen on a small one).
- Diagnose with file-based logs + **real clicks** — AppleScript's
  `click menu bar item` does not fire an NSStatusItem's button action.

### Continuously animated menu-bar icons need NSStatusItem, not MenuBarExtra

**Symptom:** SwiftUI `MenuBarExtra`'s label renders as a static image updated only
on state changes — unsuited to continuous animation.

**How to apply:** Use `NSStatusItem` + a layer-backed `NSView` with a CAShapeLayer,
animating layer properties like `lineDashPhase` (~0.3% CPU measured). Keep
declarative panels/charts in SwiftUI hosted via `NSHostingView` in an
NSPopover/NSWindow (hybrid). Mark UI types `@MainActor`; use the target/selector
Timer variant to avoid Swift 6 capture errors. App-lifetime status views need no
timer teardown in deinit.

### Create SwiftUI popover content only when opened

**Symptom:** Eagerly creating and retaining a panel containing
`TimelineView(.animation)` kept the layout engine recomputing every frame **while
hidden** — ~12% idle CPU.

**How to apply:** Create `NSHostingController(rootView:)` in the show branch and
set `contentViewController = nil` in `popoverDidClose`. Avoid decorative
animations inside the panel — use static gauges (`Shape.trim`); motion belongs to
the menu-bar side (CAShapeLayer). Diagnose with `sample <pid>`: SwiftUICore
LayoutEngine dominating means this symptom.

### Call makeKey() on a menu-bar NSPopover right after showing it

**Symptom:** A status item click does **not** activate an accessory (LSUIElement)
app, so a popover that is merely `show(relativeTo:)`-n never becomes key and
macOS draws its material in the inactive state. Under macOS 26's Liquid Glass
that reads unmistakably as a dark, dimmed translucent sheet (measured: mean
luminance of the panel body 192 → 224 of 255). The inactive rendering became far
more visible in macOS 26, so unchanged code is observed as "it went dark one day".

**How to apply:**
- Call `popover.contentViewController?.view.window?.makeKey()` immediately after
  `show(relativeTo:)`. That alone is **pixel-identical** to `NSApp.activate` +
  `makeKey()` — `makeKey()` activates the app as a side effect, so no separate
  activate call is needed.
- The side effect is that `.transient` outside-click dismissal breaks (it is
  never trustworthy once the app is active). Adopt this only together with
  explicit global + local mouse-down monitors that close the popover yourself.
- Verify on a real machine: open it with a synthetic click, capture with
  `screencapture -x -o -R <bounds>` (the composited result, backdrop included),
  and compare mean luminance before/after. **A window-only capture
  (`screencapture -l <windowid>`) drops the backdrop and cannot judge a
  translucent material** — it produces a false "it's dark" reading.

### Don't build tree drag & drop on SwiftUI List/OutlineGroup

**Symptom:** Implemented insertion-indicator tree D&D in SwiftUI, then retracted
it. (1) `.dropDestination` only reports the pointer position at drop-commit — no
insertion line possible. (2) `DropDelegate` gives positions but **session-end
notifications are unreliable** — neither `dropExited` nor `performDrop` is
guaranteed, stray `dropUpdated` events arrive after the drop, and the insertion
line lingers; the endpoint was a watchdog timer guessing at UI state.

**How to apply:**
- If insertion-indicator reordering is a requirement, write it in
  **`NSOutlineView` (AppKit) from the start**.
- If it doesn't justify that investment, **drop the feature**.
- Companion trap: never walk the filesystem from a view body (SwiftUI re-evaluates
  bodies constantly; dragging effectively freezes). Cache the tree and update by
  revision.
- Generalization: before re-implementing "things Finder can do", ask if they're
  really needed.

### Never use SwiftPM's `Bundle.module` inside an `.app`

**Symptom:** the app runs perfectly on the machine that built it and dies at
launch — `EXC_BREAKPOINT` in `_assertionFailure`, with `NSBundle.module` on the
stack — on every other machine, including a fresh install of a notarized,
stapled release. No test, no local run, and no `spctl` check catches it.

**Why:** the accessor SwiftPM generates for a target with
`resources: [.process("Resources")]` tries exactly two paths:

1. `Bundle.main.bundleURL/<Package>_<Target>.bundle` — correct for a bare CLI
   executable, wrong for an `.app`, where the bundle is installed into
   `Contents/Resources`, not the bundle root.
2. the **absolute `.build/…/release/…` path baked in at compile time**.

Path 1 never matches an app bundle, so every successful lookup runs through
path 2 — which exists only on the build machine. The result is a defect that is
structurally invisible to local development: the more you test locally, the more
confident you get.

**How to apply:**
- Don't reference `Bundle.module` in an app target. Write a locator that
  searches `Bundle.main.resourceURL` (the `.app` layout) first, then
  `Bundle.main.bundleURL` (bare executable), then the directory holding the
  compiled code bundle (`Bundle(for:).bundleURL.deletingLastPathComponent()`,
  which covers `swift test`).
- Make the final fallback `Bundle.main`, not a trap. A missing localization
  table should degrade to the English source keys — losing translations is not
  worth killing the app.
- Keep the search order and the hit test as pure functions and unit-test them;
  the runtime call is then a thin wrapper over tested logic.
- **Add this to release verification:** move `.build/*/release/*.bundle` aside,
  then launch `dist/<App>.app/Contents/MacOS/<App>` directly. That single step
  reproduces a foreign machine. Extract the release archive to a clean
  directory and launch *that* binary for the strongest form of the check.
- Audit with `grep -l 'resources:' Package.swift` across the org — every Swift
  target declaring resources is a candidate.

*Origin: two shipped macOS apps crashed at launch on first install for every
user but the author; both had passed full test suites, notarization, and manual
QA on the build machine.*

---

### GUI apps must display their version

**Symptom:** Menu-bar apps have no `--version` equivalent; forgetting a version
display leaves users **no way at all** to know their build. Bug reports without
build identity are a real cost.

**How to apply:**
- Add the version display at scaffold time. Read
  `Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString")`
  (injected from `git describe` at build; fall back to `"dev"` outside a bundle).
- **Show it verbatim** — keep `-dirty` / `-N-g<sha>` suffixes; reporters pasting
  the exact build is more useful. Enable `textSelection(.enabled)`.
- Placement: About item for menu-style apps; directly in the panel for
  panel-style apps.
- Extract the fallback logic into a pure function and test it.

---

### MenuBarExtra pushes a height onto its content

**Symptom:** a `MenuBarExtra(style: .window)` panel renders as a header sitting
directly on a footer, with the entire body missing — or rows overlap and a
chart's fill paints over its neighbour.

**Why:** the panel does not simply grow to fit. It sizes itself to the content's
*ideal* size and hands that height back to the content. A `ScrollView` is
infinitely flexible, so its ideal height is **zero** and it collapses.
`.frame(maxHeight:)` does not help — a maximum caps a height that nothing ever
requested. A `VStack` given less height than it needs compresses its children
until they overlap.

**How to apply:**
- Put `.fixedSize(horizontal: false, vertical: true)` on the root so it asserts
  the height it actually wants.
- A `ScrollView` inside such a panel needs a **definite** height. Estimate it
  from the row counts in a pure function with a floor (never collapses) and a
  cap (never runs off the screen), and guard it with a unit test. The estimate
  need not be exact — the view scrolls.
- Better still, **bound the row count so no ScrollView is needed**; the failure
  then becomes structurally impossible rather than merely fixed.
- The same applies to any greedy view: `Spacer`, `GeometryReader`, `List`.
- Add `.clipped()` to gradient area fills. If layout is ever squeezed, a clipped
  curve beats one bleeding over the next row.

### A List's ForEach ids must be unique across the whole list, not per Section

**Symptom:** in a grouped list, ticking one item appears to tick **every item of
the same kind**.

**Why:** `ForEach(names, id: \.self)` over a value that recurs in each section
makes SwiftUI treat separate rows as one row. The stored data is correct; only
the view's identity is wrong — so model tests cannot catch it.

**How to apply:**
- Give a row an identity that includes its group: an `Identifiable` value type
  whose `id` is something like `"\(groupID)/\(itemKey)"`, iterated as
  `ForEach(items)`.
- Extract the row list as a **pure function** (`static func rows(for:) -> [Item]`)
  so the uniqueness its identity depends on can be unit-tested. SwiftUI itself
  is untestable; the material its identity is built from is not.
- Shortcut from the symptom: "acting on one affects every item of that kind" is
  almost always duplicate ids. Look at the view's identity before the data.

### Register a repeating Timer for the .common run-loop modes

**Symptom:** the display stops updating **for exactly as long as a panel or menu
is held open** — it freezes the moment someone opens it to look.

**Why:** `Timer.scheduledTimer` registers for `.default` only, and the run loop
leaves that mode while a menu or popover is being tracked.

**How to apply:** build the timer with `Timer(timeInterval:repeats:)` and add it
with `RunLoop.main.add(timer, forMode: .common)`.

### Drive elapsed-time labels with a TimelineView

**Symptom:** a freshness label such as "updated 0s ago" reads **"0s ago"
permanently** — a label added to prove nothing is stuck becomes the stuck clock.

**Why:** SwiftUI redraws only when something it observes changes. An age computed
from `Date()` at render time is evaluated when the underlying data changes, prints
"0s", and then sits there until the next change.

**How to apply:** wrap the text in `TimelineView(.periodic(from: .now, by: 1))`
so the schedule ticks that text alone; nothing else redraws with it.

### Never disable a control on a status that cannot tell "no" from "don't know"

**Symptom:** a login-item switch (`SMAppService`) that **nobody can ever turn on**.

**Why:** `SMAppService.mainApp.status` returns `.notFound` for an app that has
simply never been registered — not `.notRegistered`. Reading that as "this copy
cannot be registered" and disabling the switch makes the first attempt, which is
the only one that matters, the one the interface refuses to allow.

**How to apply:**
- When in doubt, **do not disable — offer the action and report what happened**.
  If the only way to know is to try, let the user try.
- Collapse an ambiguous status into "not enabled yet" rather than inventing an
  "impossible" state.
- **Verify the state actually changed** afterwards and say so when it did not. A
  switch that springs back with nothing said is the worst outcome.
- **Show the error in the screen where the action was taken.** Rendering it
  elsewhere makes the control look inert.
- Map an external status enum onto *what the UI should do*, not straight through.

### Ask for OS permission at the moment of intent, not at first use

**Symptom:** a notification toggle is on, nothing is ever delivered, and the app
**does not appear in the OS notification settings at all**.

**Why:** deferring `requestAuthorization` to the moment an alert fires means it is
never requested until the triggering condition happens. The OS has no record of
the app, so it is absent from the settings list — leaving a switch that cannot be
configured, cannot be verified, and stays silent indefinitely if the condition
never arrives.

**How to apply:**
- Request when the switch is turned on: the moment the user said they want it,
  the moment a prompt makes sense, and what registers the app with the OS.
- **Heal a switch left on by an earlier version** by checking at launch for
  "enabled but never asked".
- When permission is refused, say so in the UI and offer a way to the settings
  pane that undoes it — never show an on switch that delivers nothing. That
  requires **keeping** `requestAuthorization`'s `granted`/`error` in a published
  property (`{ _, _ in }` makes a refusal invisible). A refusal arrives either
  as `granted == false` or as UNErrorDomain 1 "not allowed for this application".
- Re-read `getNotificationSettings` when the settings view appears and on
  `didBecomeActive`, so the denial line clears by itself once the user flips
  the switch in System Settings. Only `.denied` counts as a denial —
  `.notDetermined` means "not asked yet"; the prompt is still to come.
- **`UNUserNotificationCenter.current()` aborts outside a `.app` bundle**
  ("bundleProxyForCurrentProcess is nil"): a bare `swift run` binary and the
  xctest runner both hit it, and xctest **does** have a `bundleIdentifier`, so
  checking the identifier does not help. Test
  `Bundle.main.bundleURL.pathExtension == "app"` and skip the center otherwise;
  that also lets off-screen renders from the test target (see testing.md) run.
- The same holds for location, calendar and other OS permissions.

### Without a UNUserNotificationCenterDelegate, no banner appears while the app is frontmost

**Symptom:** notifications **arrive in Notification Center** but no banner is
ever shown. Changing the style from "Banners" to "Alerts" in System Settings
makes them appear. Focus modes turn out to be irrelevant.

**Why:** unless a `UNUserNotificationCenterDelegate` answers `willPresent`,
macOS files a **foreground** notification silently into Notification Center and
displays nothing. A menu-bar app feels like it is never frontmost, but it is
whenever one of its own windows has focus — including **at the exact moment a
"send a test" button is pressed**. So the case you most want to verify is the
one that most reliably shows nothing.

**How to apply:**
- Implement the delegate and return `[.banner, .list]` from `willPresent`.
  Returning `.list` as well keeps a notification missed while away from the desk
  in Notification Center.
- **Install it before anything is delivered**, and hold it for the app's
  lifetime: `UNUserNotificationCenter.delegate` is a weak reference, so a
  delegate kept in a local goes away immediately and the symptom returns.
- Shortcut from the symptom: "reaches Notification Center, shows no banner" is
  almost always a missing delegate. Check that before suspecting permissions or
  focus modes.

### A change in a second ObservableObject does not reach views that do not observe it

**Symptom:** a settings checkbox updates instantly while the **menu-bar display
stays stale until the next periodic refresh**.

**Why:** when the displayed values (a model) and the choice of how to display
them (a preferences object) are separate `ObservableObject`s, changing the latter
does not invalidate a view observing only the former.

**How to apply:** have the view hold **both** of the things it actually depends on
as `@ObservedObject`. Places outside the injected environment — a `MenuBarExtra`
label, for instance — are the easiest to miss.

### A process that "looks closed" but is still running must be reclaimable

**Symptom:** after a Finder-launched one-shot job, the app was kept alive a few
seconds so its completion banner would not be cut short, while dropping out of
the Dock (`.accessory`). Double-clicking a second file during those seconds let
**the previous job's timer kill the new one.** Measured damage:
- a 700 MB extraction terminated mid-write, leaving a **truncated 543 MB file
  with no error** — indistinguishable from success
- an encrypted archive's password prompt vanished **~2.8 s into typing**
- clicking the Dock icon to keep the app took the window away ~2 s later

**Why:** a deferred `terminate` is a decision about a future you cannot see yet,
but `DispatchQueue.main.asyncAfter { NSApp.terminate(nil) }` has **no handle and
no re-check**. A process hidden from the Dock still **receives open events**, so
LaunchServices happily routes new work into one that is already under sentence.
And with no path back from `.accessory`, it keeps pretending to be gone after
being handed something to do.

**How to apply:**
- Give a deferred quit **(a) a cancellable handle** (`DispatchWorkItem`),
  cancelled by *every* path that gives the process new purpose — open events,
  reopen (Dock click), opening settings, any user interaction.
- **(b) Decide again when it fires.** The schedule-time answer describes a world
  seconds out of date. Extract the rule into a **pure function** so both
  evaluations share it and can be tested (zip-porter's `OneShotQuit.decide`).
- **"Busy" outranks everything else**, including the previous job's remaining
  banner time — that banner says nothing about the job running now.
- Requests arriving while busy belong in a **queue, not a beep**. On a Finder
  launch path that shows no window, a beep is inaudible and invisible: the work
  simply disappears.
- Check the opposite direction too: after the fix, confirm the app **still quits
  once the last job is done**. Trading a premature kill for a leaked process is
  no improvement.
- **First question whether the app needs to linger at all.** As the next entry
  shows, changing how the notification is posted removed this wind-down
  entirely — and with it, the whole class of bug above became unreachable.

### Do not keep a process alive for a notification — schedule it with a trigger

**Symptom:** A one-shot app that quits right after posting its completion banner
finds the **banner vanishes immediately**. The natural fix is to keep the
process alive while the banner is up (demote to `.accessory`, `terminate` a few
seconds later) — which is exactly the breeding ground for the previous entry.

**Why:** A notification posted to `UNUserNotificationCenter` with `trigger: nil`
is presented through `willPresent` and **belongs to the posting process**; it
dies with it. Attach a `UNTimeIntervalNotificationTrigger` and presentation
belongs to `notificationd` instead. Measured on a signed binary with
timestamped screenshots:

| Variant | Process alive | Banner |
|---------|---------------|--------|
| Immediate post, quit at presentation | 0.16 → 1.17 s | **absent at t=1.5 s** |
| trigger(0.5 s), quit once `add` returns | 0.16 → **0.57 s** | visible at t=1.5 s and t=5.0 s |
| trigger(0.1 s), same | 0.12 → **0.47 s** | visible at t=2.0 s |

The constraint was never "notifications"; it was *immediate presentation*. The
wind-down was not merely unnecessary but harmful — every failure in the previous
entry happened inside those seconds.

**How to apply:**
- Schedule completion notifications with
  `UNTimeIntervalNotificationTrigger(timeInterval: 0.1, repeats: false)` and
  **terminate as soon as the `add` completion handler returns**. There is
  nothing to wait for, and 0.1 s is imperceptible.
- What you must wait for is `add` *succeeding*, not presentation: dying before
  the XPC round trip means the notification is never registered. Resolve
  authorization inside the same chain rather than from a cached flag.
- Keep the `willPresent` delegate for the resident case — without it, no banner
  appears while your own app is frontmost.
- Verify with the real binary and timestamped `screencapture`. An entry sitting
  in Notification Center does not tell you whether a banner was on screen.
- Side effect: clicking the banner after the app has exited **relaunches it**.
  Give the click a meaning — reveal the result in Finder from `didReceive`.
- Generalization: **before hardening a workaround, measure whether the
  constraint it works around is real.** Another shape of the same API may
  remove it outright.

### Notification clicks resolve by bundle ID — resident apps must enforce a single instance

**Symptom:** Clicking a notification banner of a resident menu-bar app did not
bring the running instance forward — **a second copy of the app launched**
(two menu bar items, double polling). (status-lens, 2026-08)

**Why:** When a banner is clicked, notificationd asks LaunchServices to open
the app for the bundle identifier. On a development machine the same bundle ID
is typically registered at several paths — the dev build in the build output
directory, copies extracted for release verification, the installed copy.
LaunchServices resolves *some* registered copy, and **when it picks a
different path than the running one, that copy starts as a new process** (the
same shape as Xcode launching the DerivedData build instead). Every build
re-registers the output directory's .app, so cleaning up with lsregister is
never a durable fix.

**How to apply:**
- Give resident GUI apps a **two-layer guard**:
  1. `LSMultipleInstancesProhibited: true` in Info.plist — stops
     LaunchServices launches (notification clicks, `open`) at the LS level
  2. At startup, enumerate
     `NSRunningApplication.runningApplications(withBundleIdentifier:)`; if any
     instance other than self exists, write one stderr line and exit 0 —
     covers direct binary exec and `open -n`. Cut the decision as a pure
     function (pids in, decision out) and it unit-tests trivially
- Pass through when the bundle ID is nil (bare dev binary) — enumeration is
  impossible there in the first place
- SwiftUI `@main struct X: App` has no place to run code before its scenes:
  stored-property initializers (the `@StateObject` model, often with a
  started refresh loop) run before any `init()` body. Move `@main` to a
  small `enum Main { static func main() }` that runs the guard first and
  then calls `X.main()` — a duplicate then exits before the model ever
  starts (rolled out across 10 GUI apps, 2026-08)
- **A binary that doubles as a CLI guards only its GUI launch path.** Put
  the check after argument dispatch (in the GUI branch), never at the top
  of `main`: guarded CLI subcommands exit 0 with the "another instance"
  note while the GUI runs — a silent no-op where real work was requested,
  worst for a scheduled job (caught in nvme-lens, where a top-of-main
  guard would have blanked `sample` runs; zip-porter's pack/unpack stay
  concurrent for the same reason)
- The guard protects the **launched** side. While an unguarded old version
  stays installed, the reverse direction (the old installed copy launched
  while a dev build runs) remains unprotected — shipping the fixed build is
  part of the fix
- Verify on a real bundle via both routes: direct exec (guard message,
  exit 0) and `open` (no duplicate via the LS route). Both work without
  killing the running instance
- Side effect: to try a dev build, quit the installed instance first

### Report one result per request, not per item

**Symptom:** Opening N files selected together in Finder processed them one at a
time and announced each one. **macOS replaces a banner with the next one from
the same app**, so with three items the user can read exactly one — the rest
pile up in Notification Center. Add to that N Finder reveals, N OK clicks if
results are shown as dialogs, N destination panels if the destination is "ask
every time", and N password prompts.

**Why:** A Finder multi-selection arrives as a **single `application(_:open:)`
carrying every URL** (measured). Letting the internal "one file, one job"
structure become the reporting unit turns one act of intent into N
interruptions.

**How to apply:**
- One request (one open event, one drop) means **one progress bar, one
  question, one completion report, one Finder reveal**. Weight the bar by item
  size so the denominator covers the whole request, and label it "2 of 3 —
  foo.zip".
- **One failure must not stop the rest.** Finish the others and report "2 of 3"
  with the failures named. Any result containing a failure goes to a dialog, not
  a banner — a banner is not a place to report a failure.
- Reuse answers across the request: ask for a destination once; try the password
  already entered on the next item before prompting again.
- Put the aggregation in a **pure value type**. Which outcomes may be announced
  by a banner and which must hold the user is precisely the rule worth testing.
- If a list is truncated, **say so** ("…and N more"). A silently clipped list
  reads as a complete one.

### Standard editing shortcuts do not exist unless the main menu carries them

**Symptom:** **⌘V does nothing** in a password field, and ⌘W will not close the
window. There is no code implementing either, so there is nowhere to put a
breakpoint.

**Why:** macOS delivers ⌘X/⌘C/⌘V/⌘A/⌘Z and ⌘W to the first responder **through
main-menu key equivalents**. The text field does not interpret the keystroke on
its own. An app that builds its `NSMenu` in code (no xib) and omits the Edit menu
has no item carrying `paste:`, so the keystroke reaches nothing. Having only an
app menu and a Window menu looks complete, which is what hides it.

**How to apply:**
- When you build the menu yourself, always include **Edit (Undo/Redo/Cut/Copy/
  Paste/Delete/Select All) and File > Close**. One text field is enough to make
  the Edit menu mandatory.
- **The menu bar draws the top-level `NSMenuItem`'s own title, not its submenu's.**
  An `NSMenuItem()` with a submenu attached is an **invisible menu** however
  complete its contents. The app and Window menus get away with being untitled
  only because AppKit special-cases them (process name; `NSApp.windowsMenu`) —
  which is exactly what teaches you the wrong lesson.
- Keep `NSApp` access out of the menu-building function (split `build()` from
  `install(into:)`). **`NSApp` is nil under XCTest** and touching it traps.
  Separated, the menu can be inspected in tests and the key-equivalent bindings
  pinned automatically.
- Verify against the real menu bar: System Events'
  `value of attribute "AXMenuItemCmdChar"` confirms the binding, and the
  `length of (value of field)` after ⌘V confirms a secure field actually
  received the paste.

### Put a menu-bar icon in `button.image` — `isTemplate` is ignored inside an attributed string

**Symptom:** the menu-bar symbol looked greyer than neighbouring system items,
but only in its healthy state. `isTemplate = true` was set, so the cause was not
obvious, and switching between light and dark wallpapers changed nothing. The
image was in fact embedded as an `NSTextAttachment` inside an
`NSAttributedString`.

**Why:** `isTemplate` is honoured for `NSStatusItem.button.image` only. An image
embedded in an attributed string is drawn in whatever colour it carries. The
instruction to "follow the menu bar's colour" was therefore ignored, and the
app-context `labelColor` was burned in instead — close enough to the bar's real
colour to look like a mistake, different enough to read as grey.

**How to apply:**

- Put the symbol in `button.image` and the text in `button.title` /
  `attributedTitle`, arranged with `imagePosition`. Do not mix images into text
  attachments.
- For states whose message *is* the colour (orange for warning, red for
  critical), set `isTemplate = false`: a template is recoloured by the menu bar,
  which would discard exactly that colour.
- **Do not light up green for healthy.** A colour that is lit 99% of the time
  teaches the eye to skip the icon, and then the one time it matters nobody
  notices. Let healthy take the bar's own colour and reserve colour for
  attention.

### Verify SF Symbol names resolve — a name that does not exist degrades in silence

**Symptom:** `internaldrive.fill.badge.exclamationmark` was used as an icon name.
It reads perfectly plausibly and **does not exist**.
`NSImage(systemSymbolName:)` returns nil, the fallback glyph is drawn, and the
menu bar quietly becomes meaningless.

**Why:** SF Symbols naming looks systematic but the available variants differ per
symbol — `externaldrive.fill.badge.exclamationmark` exists while the
`internaldrive` equivalent does not. It compiles, raises nothing at runtime, and
throws no exception.

**How to apply:**

- Expose **every symbol name the renderer can emit as an array and assert in a
  test that each resolves**. `NSImage(systemSymbolName:accessibilityDescription:)
  != nil` is the whole check.
- That requires a test target for the executable as well. Even with the logic
  pushed into a core library, choosing symbol names stays in the view layer.
- Either make the fallback visibly broken, or guarantee by test that it is never
  reached. A harmless-looking fallback is how this stays hidden.

### Do not fix a view's size against today's content

**Symptom:** a menu-bar app's panel, its chart internals and its settings window
each got a size measured against the content at the time (`frame(height: 560)`
and friends). Adding sections and graphs afterwards broke the display **three
separate times** — content cropped, elements compressed, text overflowing.

**Why:** a fixed value is only correct for the content in front of you when you
write it, and content grows. The failure modes — cropping and compression — look
like anything but a layout constant. And a dimension written in two places (the
view's own height, and the height the container gives it) drifts the moment one
of them is updated.

**How to apply:**

- **Declare a floor and an ideal, not a fixed size**:
  `frame(minHeight:idealHeight:)` plus a `.resizable` window. Growth then does
  not break it, and the user can widen it.
- Dimensions that must agree get **one constant, referenced from both places**.
- A view that reserves rows for text should stop reserving them when the caller
  already displays that text. Reserving unused space starves the part that
  matters — a chart given 56pt with 46pt of reserved text had 10pt of plot.
- The only fixed sizes worth keeping are the ones a container legitimately owns,
  such as a popover's width.

### Apple Mail multi-message drags arrive only via the pre-10.12 file-promise protocol

**Symptom:** A drop target registered for
`NSFilePromiseReceiver.readableDraggedTypes` accepted single-message drags from
Apple Mail but rejected multi-message drags outright — the drag never matched.

**Why:** Sampling the drag pasteboard (`NSPasteboard(name: .drag)`) shows that a
single-message Mail drag carries both the modern promise types and the
pre-10.12 legacy protocol (`com.apple.pasteboard.promised-file-url` /
`NSPromiseContentsPboardType`), while a **multi-message drag carries only the
legacy protocol** — the modern types vanish from the pasteboard entirely. On
top of that, `receivePromisedFiles`' reader block frequently never fires for
Mail (a known platform bug).

**How to apply:**
- Register `com.apple.pasteboard.promised-file-url` in
  `registerForDraggedTypes` and resolve it inside `performDragOperation` via
  the deprecated `namesOfPromisedFilesDropped(atDestination:)` — the only API
  that keeps Mail's multi-message promise, and it returns the **exact count of
  promised file names** (which also speeds up single-drop completion).
- The files are written asynchronously. Decide completion by "promised count
  reached and every file's size stable for a window", with a hard deadline
  that **always** produces an outcome (zero files at the deadline is a visible
  failure, never a silent timeout).
- Give every drop its own unique temp subdirectory: name collisions,
  back-to-back-drop races, and directory-diff snapshots all disappear.
- For diagnosis, a small sniffer that polls the drag pasteboard every 100 ms
  and prints the type list plus
  `canReadObject(forClasses: [NSFilePromiseReceiver.self])` settles the
  question decisively. No drop needed — start a drag and cancel with Esc.

## Wails (Go + WebView)

### window.alert() does not reliably appear

**Symptom:** In Wails v2 (macOS WKWebView), `window.alert()` / `confirm()` /
`prompt()` silently do nothing. The whole error-notification path was dead, but
error paths are rarely exercised, so nobody noticed for a long time.

**How to apply:** Route user notifications through a frontend helper → Go
binding → `wailsRuntime.MessageDialog` (Info/Error). Use inline UI for
confirmations (two-click delete, N-second confirm). Emitting result notifications
from the Go side keeps the frontend to fetch-and-refresh. Never trust GUI error
display until you've **deliberately triggered an error on the real app**.

### Never run blocking work synchronously in OnStartup

**Symptom:** Spawning child processes and probing a container engine synchronously
inside `OnStartup` delayed/hung startup. Worse, **Wails dispatches frontend
binding calls concurrently while OnStartup is still running**, so frontend init
raced against un-constructed backend state.

**How to apply (the non-blocking startup shape):**
1. Fix the window size at creation (`options.App{Width,Height}`).
2. Constructors return immediately; move external-dependency init to a
   `StartBackground(ctx)` goroutine.
3. Two-stage readiness signals (`app:ready` / `tools:ready`) via EventsEmit, plus
   query bindings to survive missed events (emitted before listeners attach).
4. Gate the input UI until `tools:ready` (kills the send-to-half-initialized
   race).
5. Always give probes a timeout (`context.WithTimeout`).
6. Fields shared with goroutines get a mutex + `go test -race`.

### Without explicit Menu and mac.About, standard menus/About vanish

**Symptom:** With no `Menu:` set, macOS shows a minimal default menu with no About
item, and `Cmd+C/V/Z` may not work as native shortcuts. Shipped that way for a
long time, leaving users unable to check the version from the GUI.

**How to apply:** At scaffold time wire both a `Menu:` built from
`menu.AppMenu()` + `menu.EditMenu()` + `menu.WindowMenu()` and `Mac.About`
(`*mac.AboutInfo` with title / version string / icon). To call runtime APIs from
menu handlers, the ctx doesn't exist at menu-construction time — reference the
ctx captured at startup via a closure.

### Replacing appicon.png does not regenerate the Windows icon.ico

**Symptom:** macOS `.icns` is regenerated from `appicon.png` on every build, but
`build/windows/icon.ico` is generated once at `wails init` and never again. After
adopting a custom icon, the Windows .exe shipped with the default "W" logo.

**How to apply:** Regenerate `icon.ico` whenever the icon changes. Pillow works:

```python
from PIL import Image
src = Image.open('build/appicon.png').convert('RGBA')
src.save('build/windows/icon.ico', format='ICO',
         sizes=[(256,256),(128,128),(64,64),(48,48),(32,32),(24,24),(16,16)])
```

### Put multi-platform release targets in the Makefile from day one

**Symptom:** A Makefile that only ran `wails build` forced manual cross-builds and
renames at release time, producing hand-work inconsistencies ("only the Intel
bundle had a different name inside the zip").

**How to apply:** Scaffold `build-darwin-arm64` / `build-darwin-amd64` /
`build-windows-amd64` / `build-all` / `package: build-all`. Each per-arch build
starts with `rm -rf build/bin` to avoid re-signing stale apps into packages.
Package notarizes + staples, then renames to canonical names in a staging dir
before arch-suffixed zipping. windows/amd64 cross-builds fine from Apple Silicon
with Wails v2.12.

### Avoid translucent windows

**Symptom:** `WebviewIsTransparent: true` + CSS rgba backgrounds looked
"native-ish" but long text became hard to read over the desktop, and true blur
requires private APIs.

**How to apply:** Start with `WebviewIsTransparent: false`. Surface-level CSS
tokens use `rgb()`; layer tokens (sitting on opaque parents) may stay rgba.
`TitlebarAppearsTransparent: true` is fine (within native behavior).

### go-duckdb needs the no_duckdb_arrow tag

**Symptom:** Embedding go-duckdb in Wails fails with Arrow CGO link errors
(`Undefined symbols: ArrowArrayIsReleased`).

**How to apply:** If only the `database/sql` interface is used, exclude Arrow
with `wails build -tags no_duckdb_arrow`. Record it in the Makefile.

---

## Shared patterns

### Embed CLI subcommands in the GUI binary (single binary)

**Symptom:** Adding a background CLI mode as a separate binary complicates build
targets, .app bundling, and path resolution.

**How to apply:** Check `os.Args[1]` in `main()` and route before `wails.Run()`.
In CLI mode `wails.Run()` is never called, so there is zero GUI overhead. The app
can spawn itself via `os.Executable()`.

### Measure a framework error type's full case table before you design error UX

**Symptom:** a translation app showed one line for every failure —
`"Couldn't translate — the language model may still be downloading. "` plus
`error.localizedDescription`. An unsupported language pair, an internal service
fault and a genuinely missing model all rendered identically, and the
"may still be downloading" half was wrong in most of them.

**Why:** `localizedDescription` is written for the framework's own system
dialogs, not for your UI. Measured against the macOS 26.5 SDK, `TranslationError`
returns **`"Unable to Translate"` for seven of its eight cases** (only
`nothingToTranslate` differs), and every case bridges to `NSError` domain
`Translation.TranslationError` **code 1**. The distinguishing text lives in
`failureReason`, which nothing was reading. Assuming `localizedDescription` is
informative is the whole bug.

**How to apply:**
- Before designing any error surface, write a throwaway program that prints
  `localizedDescription` / `failureReason` / `NSError.domain` / `.code` for
  **every** case. Get the case list from the SDK interface first:
  `$(xcrun --show-sdk-path)/System/Library/Frameworks/X.framework/Modules/*.swiftmodule/*.swiftinterface`.
- A `struct` error type with a custom `~=` cannot be `switch`ed on shape. Verify
  the operator actually discriminates with an **N×N match matrix** — an exact
  diagonal, and unrelated errors matching nothing. (`TranslationError` passes.)
- Put the matching in one `classify(Error) -> YourEnum`, the only place allowed
  to touch the framework type, and make the message rendering a **pure function**
  you can unit-test.
- Phrase messages in three registers: the cause, the fix **naming the control
  that applies it** (upstream strings cannot — they don't know your UI), and a
  selectable technical tag for bug reports.
- **Never drop the upstream text.** An unrecognised error must carry
  `failureReason ?? localizedDescription` plus domain/code, or the original bug
  returns for exactly the cases you failed to anticipate.
- **Never hardcode a guessed cause as a prefix.** A fixed "it might be X" is
  actively misleading in every case where it is not X.

### A state where you deliberately do nothing must still be nameable in the UI

**Symptom:** a translator was reported as "you can't tell whether it's working."
It had accumulated five correct decisions to withhold a translation — an open IME
composition, input too short to identify, a 600 ms debounce, source equal to
target (echo), and a first-use model download — and **every one of them was
silent**. A correct wait is indistinguishable from a hang.

**Why:** the app already had `isTranslating: Bool`, and no view read it. Even
reading it would not have fixed this: a boolean cannot say *why* nothing is
happening, and "why" was the whole question.

**How to apply:**
- Model state as an `enum Phase`, not `isLoading: Bool`. If you need a boolean,
  derive it from the phase (`isWorking: phase == .preparing || .translating`) so
  there is no second source of truth to drift.
- Map phase → presentation (symbol / text / spinner / tone) in a **pure function**
  and unit-test it. One test asserting *every* case returns non-empty text stops
  a future phase from being added silently.
- **Any early `return` that quietly skips work sets a phase first.** That is the
  reviewable rule: look for a state assignment above each early return.
- Say the way out, not just the state: "Can't tell the language yet — type more,
  or pin it on the left."
- Render the status row **unconditionally** so the layout never jumps, and put
  the spinner and the icon in one shared slot so text does not shift sideways.
- **Do not spin for a wait.** A spinner claims progress; held and pending states
  get a static icon.

This is the same principle as showing the version string in the panel: for a
menu-bar app with no menu, no About item and no log file, **what is on screen is
the entire information channel** — for state, and for the technical cause of a
failure.

### CLI stderr warnings never reach the GUI — put deliberate "$0 / skipped" states into the JSON contract so the UI can name them

**Symptom:** A menu-bar app that is a thin front-end over a usage-accounting CLI
kept showing a new model's turns as $0 (twice, for different models, 2026-07
and 2026-09). The CLI had the right design — "a model missing from the rate
table is stored at $0 and a warning goes to stderr" — but the app calls
`ingest` every minute, the exit code is 0, and nobody reads stderr. All the
user saw was a complete-looking number that was silently too small.

**Why:** "A deliberate hold-off state must name itself in the UI in the same
commit" (above) breaks at a process boundary. The CLI author considers the duty
done once the warning is printed; the GUI author reads only JSON. A warning
emitted at event time (the moment of ingest) is gone unless the GUI is present
at that moment. And after an app update ships a newer rate table, the rows the
old build stored at $0 are still there — the state lives in **stored data, not
in an event**.

**How to apply:**
- Make every deliberate hold-off state (unpriced, skipped, partial failure) a
  **field of the JSON the GUI already reads**. stderr, logs and exit codes are
  not a contract.
- **Derive the state from stored data** (e.g. "rows with tokens but zero
  cost"). Event-time counters vanish on restart and on update; a row-derived
  count is right every time, including the "updated but not yet repriced"
  case. Deriving it without the rate table keeps it independent of
  configuration.
- Make the fields optional on the GUI side and treat absence as healthy, so an
  older CLI still works.
- In the UI, name the count and the subject, and **put the exit in the same
  box** (here a "Reprice" button that runs the CLI's recomputation from the
  GUI). If the state survives that exit, switch the wording to the next exit
  (update the app) and disable the button.
- Mark the always-visible number too (the menu-bar figure). For a user who
  never opens the popover, that mark is the only way to say "this number is
  incomplete".
