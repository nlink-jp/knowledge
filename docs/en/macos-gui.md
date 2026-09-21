# macOS GUI Apps

Traps and established patterns from developing native SwiftUI / AppKit apps and
Wails (Go + WebView) apps. Each entry follows **symptom → why → how to apply**.
For signing and notarization, see [release-engineering.md](release-engineering.md).

---

## SwiftUI / AppKit (mostly menu-bar resident apps)

### Separate macOS 27 menu bar API discovery from functional proof

**Symptom:** A menu bar organizer feasibility probe resolved the private
`MenuBarClientCore` framework and its hiding-related classes and methods, but a
restricted execution environment saw neither displays nor MenuBarAgent. Both
became visible in desktop context. After the user granted the signed diagnostic
app Accessibility access, reading AX children also succeeded. No hiding,
movement, or restoration had actually been exercised. (2026-09)

**Why:** Framework loading, GUI-session access, TCC Accessibility trust,
individual item identity, and successful control are separate conditions. An
empty result does not distinguish them. macOS 27's public `NSStatusItem` session
API manages an app's own panel, not arbitrary other applications' items.

**How to apply:**
- Report API resolution, permission, AX reachability, individual item extraction,
  hiding/movement, and restoration separately, with OS build and execution context.
  Never label an unexecuted step as successful.
- Distinguish failed, empty, and partial reads. Failure does not mean every item
  has been successfully hidden.
- Rerun the unchanged signed diagnostic app after the user grants access.
  Rebuilding can change the permission test conditions; do not mix those results.
- Matching class names and argument counts do not establish invocation success.
  Validate actual argument types, including Objective-C object `@` versus block
  `@?`, and results before using a private call.
- Check unrelated icons and their functions during a hiding test. The reference
  reports collateral behavior including Notification Center; reproduce on the
  intended OS before making an adoption decision.

**Follow-up observation (2026-09-15):** Two bounded workers on build 26A428
confirmed activation and AX-label return after normal release and SIGKILL.
However, an explicitly allowed fixture B and the audio/video system identifier
were absent alongside the target during both requests. Refreshing fixture
registration and restarting did not resolve B; its cause remains unknown. This
is a one-window AX result, not a general compatibility or pixel-visibility claim.
The updated probe also reported AX trust false while Settings showed its toggle
on; an independent authorized observer supplied the observations.

- Preserve targets, allowed control apps and unrelated system identifiers in
  before/during/after records. Target disappearance alone is insufficient.
- Timestamp observations against the actual worker lifetime. Treat menu opening,
  action execution, visible appearance and full system function as separate checks.
- A visible permission toggle is not proof that a newly signed process is trusted.
  Report the mismatch; never convert an unreadable tree into a successful test.

References: [Apple's own-status-item API discussion](https://developer.apple.com/videos/play/wwdc2026/289/),
[macOS 27 reference implementation limitations](https://github.com/fif7y/pelmet/blob/2a1968cc66816b02d7900d9efb13e7119642309b/docs/FAQ.md).

### Menu bar spacing: distinguish fresh-app geometry from global live behavior

**Symptom:** On macOS27.0(26A428), changing current-user/current-host global
`NSStatusItemSpacing` and `NSStatusItemSelectionPadding` together to4/4 and24/24
changed freshly launched AppKit fixture window widths. Text measured37→25→45→37
points across original,compact,wide,restored states. A human confirmed visible
change in a longer trial. Finder was not restarted and the user did not log out.

**Why:** New processes can consume settings even when a global live update is
not established. Own NSStatusItem geometry, actual screen appearance, other
applications, and system items are different pieces of evidence. Both keys were
changed together, so their independent effects remain unproven.

**How to apply:**
- Treat this as a demonstrated fixture mechanism, not an all-app promise. Fresh
  launch sufficiency does not establish that relaunch is necessary.
- Preserve exact key absence and values in the precise preference scope. Do not
  replace absence with an assumed OS default; record any-host fallback separately.
- Serialize experimental writes and watchdog restoration. Once recovery begins,
  reject later experimental writes even if restoration fails. Use unique payloads.
- Measure settled samples, retain human visual feedback separately, and restore
  and re-read original preferences with a fresh process.

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
  `makeKey()`, so no separate activate call is needed.
  - This bullet used to give the reason as "`makeKey()` activates the app as a
    side effect". That was an **inference** from the identical rendering; the
    activation state itself had not been measured. Measured on macOS 27.0, the
    app does not become frontmost after `makeKey()` (next entry; same result
    in two apps). If anything it goes the other way: an accessory app that was
    frontmost because its settings window had been opened was no longer
    frontmost 0.15 s after a status item click opened the popover — the
    previously frontmost app was, 3 times out of 3, with or without
    `makeKey()`. The drawing benefit stands as a separate fact.
- Do not leave outside-click dismissal to `.transient`: ship this **together
  with explicit global + local mouse-down monitors**. That is not because of a
  side effect of `makeKey()`, though — see the next entry.
- Verify on a real machine: open it with a synthetic click, capture with
  `screencapture -x -o -R <bounds>` (the composited result, backdrop included),
  and compare mean luminance before/after. **A window-only capture
  (`screencapture -l <windowid>`) drops the backdrop and cannot judge a
  translucent material** — it produces a false "it's dark" reading.

### Don't leave a menu-bar NSPopover's outside-click dismissal to `.transient`

**Symptom:** With a `behavior = .transient` popover open, clicking an empty
stretch of the menu bar, or another menu-bar app's panel, does not close it.
Clicking another app's window does, so an everyday check concludes "it closes".
(load-spinner, 2026-09, macOS 27.0)

**Why:** As measured, `.transient` closed the popover **only when the outside
click landed in a window that takes activation** — another app's normal window
(even when that app was already frontmost). Clicks on **surfaces that take no
activation** — an empty stretch of the menu bar, another process's
non-activating panel — were missed 3 times out of 3.

The defect had been handed over as "`makeKey()` activates the app, and
activation breaks `.transient`". **Neither half reproduced.** A control build
with only the `makeKey()` line removed behaved identically, and the app never
became frontmost after `makeKey()` (`NSWorkspace.frontmostApplication` and
`lsappinfo front` agreed, sampled from 0.15 s after opening). A different app's
observation — dismissal stopped working after a settings window +
`NSApp.activate` was added (status-lens, 2026-08) — stands as an observation.
Measured on that same app on macOS 27.0 (2026-09), though, **activation
history changed nothing**: a control build with only the monitor call removed
gave the same numbers in all three states — never activated / settings window
open and the app made frontmost before every trial / settings opened, then
closed (windows that take activation closed 3/3; a non-activating panel and
the empty menu bar 0/3). The audit argument "this app never activates itself,
so it is unaffected" does not hold: the app that audit cleared had this very
defect.

**What `makeKey()` does to this differed between the apps.** In load-spinner
removing it changed nothing; in status-lens, removing `makeKey()` as well made
**nothing close at all** (0/3 on all four kinds of surface, including clicks
that made another app frontmost — and that was the shape of the build the
2026-08 observation was made on). All the two apps had in common: `makeKey()`
is not what breaks dismissal, and the monitors are needed either way.

**How to apply:**
- For a menu-bar `NSPopover`, **unconditionally** install
  `NSEvent.addGlobalMonitorForEvents` + `addLocalMonitorForEvents`
  (`.leftMouseDown` / `.rightMouseDown`) while it is shown and `performClose`
  yourself. Remove both in `popoverDidClose`.
- **The local monitor must ignore the status item button's window.** The
  button's own action toggles the popover, so closing here too turns one click
  into close-then-reopen. Ignore the popover's own window as well. This decision
  (where the click landed → close or not) can be extracted into a pure function
  and pinned by a test. **On macOS 27 the same click also reaches the global
  monitor, so this alone is not enough** — see the next entry.
- The local monitor **also closes on a mouse-down in any other window of the
  same app**. When you add a separate window, or a control that opens a window
  of its own, reconcile it with that rule in the same change.
- Swift 6: monitor handlers are nonisolated. Wrap the body in
  `MainActor.assumeIsolated` and keep the non-Sendable `NSEvent` out of its
  return value (read `event.window` outside, `return event` outside).
- **When a defect is handed over as "X causes it", build a control with X
  removed and check whether the symptom persists before fixing anything.** A
  few minutes of control experiment kept a false causal claim out of the docs
  and the commit history here.
- **Do not carry a control build's result over to another app.** "Removing
  `makeKey()` changes nothing" was a fact about the first app; in a second app
  built from the same parts, removing it meant nothing closed. The rule
  (install the monitors) transfers; the causal detail has to be re-measured in
  that app before it is written down.
- Verify on a real machine with the procedure in "A menu-bar app's popover can
  be verified from a script" below. Aim outside clicks only at **a window or
  non-activating panel you own**, or at a point whose AX role you have just
  re-read (`AXMenuBar`). **Testing only a click on a normal window passes a
  broken app** (the first attempt here did). Cover: normal window / the
  already-frontmost app's window / non-activating panel / empty menu bar / an
  inside click keeps it open / a second button click closes it without
  reopening.

### A click on your own status item also reaches the global monitor (macOS 27)

**Symptom:** With the panel open, clicking the status item again does not close
it. It vanishes and comes straight back (gone 23–40 ms after mouse-down, shown
again 68–221 ms after it — with the button still held). Outside clicks close it
correctly, so a check that only opens the panel and clicks elsewhere never
notices. (nvme-lens, 2026-09, macOS 27.0)

**Why:** On macOS 27 the menu bar is hosted by another process, so a click on
your own status item reaches the **global mouse-down monitor first** (in every
trace taken, whether or not the app was active) and the button's action 14–49 ms
later. If the monitor closes unconditionally and the action toggles on
`isShown`, one click becomes close-then-reopen. The previous entry's rule — the
local monitor ignores the status item's window — does not prevent it: the same
click arrives on the global side too.

Whether it surfaces was **decided by the close animation**. With
`animates = false` the monitor's close completes at once and the action finds
`isShown == false` and reopens. With the default animation `isShown` is still
true when the action arrives, so it "closes again" and looks right (6/6 on a
control build; two other apps with the same structure, 3/3). That is a timing
margin, not a guarantee.

The two events **cannot be matched by identity**. The action runs under a
synthesized `leftMouseUp` whose `eventNumber` was 0 whatever number the
mouse-down carried (checked by numbering the synthetic clicks), and the
timestamps differ. And while the app is active (after a click inside the panel)
**the action sometimes never arrives** (3 of 5 traced re-clicks) — so a fix that
makes the monitor skip clicks on the item and leaves the closing to the action
closes nothing in that state. It is also why re-clicks in that state sometimes
"worked" before the fix.

**How to apply:**
- **Match the two by order.** Keep closing on every global mouse-down; when the
  click was on your own status item, note that this click's action is awaited.
  The next action consumes the note and does nothing. If no action comes, void
  the note at the next mouse-down the monitor sees — so keep the monitor
  installed "while the panel is shown, or while an action is awaited", and run
  the same sync from `popoverDidClose`. Rely on none of: a time window, the
  animation's margin, `isShown` alone. Extract the state machine as a pure value
  type and pin it with tests.
- Take the note **only for clicks on the item**. Where item clicks never reach a
  global monitor, no note is ever taken and the behaviour stays the plain toggle
  it was (no regression).
- Decide "on the item" from **the button's window frame, not the button's, read
  at click time**. Measured: the window is the menu bar's full height (30 pt),
  the button 22 pt, and the rows between belong to the item. A variable-length
  item changes width with its content, so a frame read earlier is stale within
  tens of seconds.
- Judge the edges with the **top-left pixel convention**. A global monitor's
  event has `window == nil`, so `locationInWindow` is already in screen
  coordinates (bottom-left origin), and **the screen's top row is exactly
  `frame.maxY`** and belongs to the item (where a pointer pushed against the
  edge sits). `frame.minY` (the first row under the menu bar) and `frame.maxX`
  (the neighbour's first column) do not. `CGRect.contains` answers the opposite
  on both vertical edges. Measure which points belong to the item by clicking
  them with the panel closed and seeing whether it opens, and pin those boundary
  points in tests.
- Always include **a re-click of the status item** in the verification cells,
  plus **a re-click after a click inside the panel** (app active) and **that the
  click after it opens the panel**. Try the item's top row, bottom row, left and
  right columns, and a right-click. The popover's arrow overlaps the item's
  bottom rows at its centre; a click there is a click inside the panel (neither
  the monitor nor the action fires) — not a defect.
- **Remove the monitor in one place, and drop the reference there.**
  `NSEvent.removeMonitor` over-releases a monitor it is handed twice. The first
  version that synced the monitor from `popoverDidClose` **segfaulted when the
  app was quit from the button inside the panel**: `applicationWillTerminate`
  removed the monitor and kept the reference, termination closed the panel's
  window, and `popoverDidClose` removed it again. Fail a test when a second
  removal site appears (counting call sites in the source is enough).
- Put **the ways out of the app** among the verification cells: quitting from a
  button inside the panel, quitting with other windows open, an external
  terminate while the panel is open — judged by **exit status and new crash
  reports**. A probe that only ever terminates the app from outside with the
  panel closed never runs the panel's close during termination (here a human
  hand check found it, not the probe).
- Give synthetic clicks a `mouseEventNumber`. Without one every click is number
  0, and the question "can they be matched by number?" gets a false answer
  either way.

### Put a menu-bar panel with controls in a non-activating panel — activating on open does not fix it

**Symptom:** a pop-up button (SwiftUI `Picker(.menu)`) inside a menu-bar popover: on the
**first click after opening**, the menu flashes and disappears. From the second click on it
opens. Reported as "only the first click behaves erratically". (net-meter, 2026-09, macOS 27.0)

**Why:** `makeKey()` makes the popover key but leaves the accessory app inactive (see "Call
makeKey() on a menu-bar NSPopover right after showing it"). So the first click inside the panel activates the app, and that notification
arrives **while the menu the click just opened is tracking** and ends the tracking. In the
trace the menu ended 78 ms after it began, with `didBecomeActive` right behind it.

**This entry once recommended the wrong remedy (corrected 2026-09-20).** It said: call
`NSApp.activate` right after showing; measured, activation completes 12–39 ms later (nine of
nine). Right after that shipped, the report was "only the first time after launch, the menu
does not open". Recorded with the signed bundle launched through LaunchServices, **the
activation request was not honoured right after launch** — the frontmost app stayed the
previous one, no `didBecomeActive` arrived, the first click then activated the app and the menu
ended after 71 ms. That refusal is a fact this file already records under "Two pitfalls of a
menu-bar NSPanel". The nine-of-nine check used a build started from a terminal — a child of the
frontmost app — and never met the condition.

**How to apply:**
- Host a panel with controls in an **`NSPanel` with `.nonactivatingPanel`**, and never ask for
  activation. A click in the panel does not activate the app, so nothing interrupts menu
  tracking. Under the same condition (the first click, 21.8 s after launch) the menu stayed
  open for 1,906 ms and no activation happened at all (one run). Nothing is taken, so nothing
  has to be handed back on close. Follow "Two pitfalls of a menu-bar NSPanel" and the existing
  implementations (task-clock-gui, instant-translate, net-meter).
- **Test anything that depends on activation with the signed bundle launched through
  LaunchServices, within 30 seconds of launch.** A build started from a terminal or an IDE is
  a child of the frontmost app and the OS treats it differently. "N out of N" means something
  only within the conditions that were tried.
- A hidden title bar still has a safe area. SwiftUI content in a `.titled` +
  `.fullSizeContentView` panel asked for 32 pt more height (535 pt against 503 pt).
  `ignoresSafeArea()` on the view did not change the size asked for; `safeAreaRegions = []`
  on the hosting controller did.
- While a non-activating panel is key, `NSApp.isActive` **reads true** (no `didBecomeActive`,
  another app still frontmost; one run). For "is the app active", ask
  `NSWorkspace.shared.frontmostApplication`.
- When adopting a mechanism, search the knowledge base by the **mechanism's name** (here:
  activation). Searching by the symptom or the part (popover, Picker) did not reach the entry
  that records the refusal.

### Nothing tells a non-activating panel that the user went elsewhere — close on the ways that involve no click, too

**Symptom:** a menu-bar panel moved from `NSPopover(.transient)` to an `NSPanel` with
`.nonactivatingPanel`, with outside clicks closed by global + local mouse-down monitors. The
independent review before the release pointed out that it no longer closed on **moves that
involve no mouse-down**: after Cmd-Tab the panel kept floating over the other app, and after a
Space change it stayed open on the old Space, out of sight. The next click on the item then
only closed a panel nobody could see — one dead click, from the user's side. (net-meter,
2026-09, macOS 27.0. The popover had closed itself in both cases.)

**Why:** the app never becomes active, so no `didResignActive` ever arrives. The panel gets
the events addressed to it and nothing else; no notification says the user has gone somewhere
else. Mouse-down monitors watch only one of the ways of going elsewhere.

**How to apply:**
- With one close path in place, **count its entrances by kind**: an outside mouse-down, a
  re-click on the item, Esc, `NSWorkspace.didActivateApplicationNotification` (an app other
  than yours came forward) and `NSWorkspace.activeSpaceDidChangeNotification`. The first four
  were confirmed on hardware (one to four times each); the Space change was not measured.
- Do not close on `didResignKey`. On an item click it runs before the global monitor, and
  close becomes close-then-reopen.
- When moving away from a popover, **list what the popover was doing implicitly before removing
  it**: placement, material, outside clicks, closing on app and Space changes, following the
  item when its width changes. A rehosting done to fix a reported defect tends to drop
  behaviour nobody reported.

### Right after a status item's length changes, its window has the new width at the old origin — follow it by the move notification

**Symptom:** a setting inside the panel changes the item's width (`NSStatusItem.length`), so the
panel was placed below the item again — one run loop turn later, on the assumption that the
window "has not moved yet". The panel ended up as much as 57 pt off: the same display mode put
it at x=2102 on open and at x=2159 after coming back from another mode (twice). (net-meter,
2026-09, macOS 27.0)

**Why:** setting `length` resizes the item's window **at once, width only**; its origin stays
where it was, so its right edge is wrong. The process that hosts the menu bar corrects the
origin **29–41 ms later**, and `NSWindow.didMoveNotification` follows (three changes out of
three). The frame read on the next run loop turn was that in-between state.

**How to apply:**
- Whatever follows the item is placed again from the item window's `didMoveNotification`
  (register with `object: nil` and check identity in the handler: your own panel's `setFrame`
  posts the same notification, and without the check it loops). After the change: three mode
  changes, one placement each, 44–50 ms after the change, at the item's centre every time.
- Do not solve it with a delay. Putting "100 ms" where "one turn" failed is the same guess with
  a bigger number.
- The item also moved by 1–5 pt with nothing of ours happening (a neighbour changed width). The
  move notification is needed for more than your own setting.
- How to measure: give the diagnostic build — and only it — a hook that cycles the setting and
  records the item window's frame with timestamps. It needs no click and no panel and can be
  repeated at will. Ask the user only for checks that really need a person's hands.

### A panel that refreshes every second must not refresh while one of its menus is tracking

**Symptom:** in the same panel the menu opens and an item can be picked, but **the choice does
not take**. The binding's setter received the **old value** (three times out of three).

**Why:** the panel's model was updated once a second. When an update lands during menu
tracking, SwiftUI **re-syncs the pop-up button to the current value**, and the item picked
afterwards is reported as that old value. All three times, at least one update fell inside
the tracking.

**How to apply:**
- Hold updates to the panel from `NSMenu.didBeginTrackingNotification` to
  `didEndTrackingNotification` and deliver one afterwards. Count depth — menus nest. Deliver
  it on the next run loop turn, so the pop-up button's own action runs first.
- The gate (begin, end, "may I update now?") is a small pure type and can be tested.
- Do not hold back the menu bar item itself; only the view that owns the menus.
- Measured after the fix: seven selections of a different value, seven applied.
- The setter runs **about 190 ms before** `didEndTracking` is posted. A script matching a
  trace that looks only after the end notification counts good selections as lost.

### With the default animation, `popover.isShown` stays true for about half a second after a close is requested

**Symptom:** clicking the status item repeatedly, the panel very often fails to open.

**Why:** open/close was decided from `popover.isShown`. In the trace, closing within about
half a second of opening took **534–546 ms** (four times) from `performClose` to
`popoverDidClose`, with `isShown` true throughout. The panel looks gone, so the user's next
click means "open", the app reads "close", and nothing happens — at two clicks a second, every
other open click was lost. The re-click guard in place at the time, "`isShown` plus a 0.25 s
window", was exactly the shape "A click on your own status item also reaches the global
monitor" warns against, and it failed as predicted.

**How to apply:**
- Replace it with the order-matching toggle of that entry and set
  `popover.animates = false`. A close then takes 2–18 ms, and 58 clicks at a median of 183 ms
  apart alternated correctly in 57 of 57 transitions.
- Never decide open/close by a time window, by the close animation's margin, or by `isShown`
  alone.

### A bar in a small scrolling graph is a sample — and motion has to be looked at as a filmstrip

**Symptom:** a small bar graph in the menu bar should be moving from right to left, but watched
over time its segments changed oddly.

**Why:** when the bars were made wider, each became a five-second time bucket to keep the
window at about a minute. Drawn second by second and stacked as a filmstrip, the graph stood
still for about five seconds, then jumped a whole bar, and in between only the rightmost bar
kept changing height (it shows the bucket's maximum). And whenever a peak entered or left the
window, every bar rescaled in the same instant. The version before that measured buckets back
from `now`, so each sample crossed a bucket edge at a moment set by its own phase and two
bursts sat one column apart on some ticks and two on others. One-second buckets would still
lose and merge samples whenever a jittery timer straddles an edge.

**How to apply:**
- Do not cut bars by time: draw **the last N samples, one bar each**. The graph moves by
  exactly one bar per sample and a bar never changes once drawn. There is no edge to fall
  across. Leave the longer view to another surface (the panel's chart).
- Auto-scale **up at once and down gradually** (20% per sample, say). Bars already drawn all
  jumping taller the instant a peak leaves reads as the past being rewritten. Keep the state
  in the controller, not the renderer; move it only when a new sample arrives; start fresh
  when what is displayed changes.
- **Motion cannot be judged from one rendered frame or from unit tests.** Feed a fixed input
  through the real controller and renderer and write the frames, one per second, stacked into
  one image. An independent review and every test had missed this.
- What can be pinned, pin: "every frame equals the previous one shifted by one bar".

### Things that must look aligned are placed from one source, and the offset is measured in ink

**Symptom:** in a two-line menu bar item, the arrows and the numbers beside them looked slightly
off-centre from each other.

**Why:** the arrows had constants of their own ("2 to 10 pt above the row's bottom") while the
digits sat where the font put them. The upward arrow happened to match; the one whose tip
points down had the vertical middle of its ink **1.00 pt** above the digits' — exactly one
pixel on a 1x display.

**How to apply:**
- Place the shape from the font's metrics (baseline and cap height) so it spans the digits'
  band. Do not keep two sets of constants that happen to agree.
- "Looks aligned" can be measured offscreen: take the middle of each one's ink from its top and
  bottom extent and hold the difference within 0.5 pt, at 1x and at 2x (1x displays exist).

### A status item's button reports the menu bar's own appearance, not the system's

**Symptom:** a menu bar item is to be drawn in colour, but a template image cannot carry colour,
and without templating the foreground colour is yours to choose.

**Why:** measured: with the system in light mode and the app's appearance Aqua, the status item
button's `effectiveAppearance` was `VibrantDark` (a dark wallpaper makes the menu bar dark; one
observation).

**How to apply:**
- Default to a template image drawn in black with alpha: the OS picks the colour and the item
  matches the system's own.
- Only for a coloured finish, turn templating off and choose the foreground from
  `button.effectiveAppearance`. Do not look at `NSApp.effectiveAppearance` or the system setting.
- Give the image both a 1x and a 2x representation, each aligned to its own pixel grid. With
  one, a Mac with displays of mixed scale shows an interpolated, blurred item on one of them.
- With drawing as a pure "values + finish → image" function, appearance × scale × state can all
  be checked offscreen.

### Size a field for the longest real value, not for the sample data

**Symptom:** the panel cut IP addresses in the middle.

**Why:** the layout had only ever been looked at with `2001:db8::10`. In a "label column + value
column" table the value had 158 pt; a typical global IPv6 address (36 characters) needs 232 pt
and one with no run of zeros to compress, 39 characters, 254 pt.

**How to apply:**
- **Measure** the longest real value before fixing a dimension; move long values out of the
  table into a row of their own across the full width.
- Pin "the worst case fits" with a test, and use the worst case in previews and layout tests.

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

### In a polling UI, never share one field between the poll's result and the action's answer

**Symptom:** the user presses Run, the daemon refuses the request, and the panel says
**nothing at all** — indistinguishable from never having clicked. The report comes back as
"my manual runs are not recorded". The error-reporting path exists in the code and works
in isolation. (task-clock-gui, 2026-09)

**Why:** each action re-polls when it finishes, to show the real state rather than an
optimistic guess. The error was written to one shared field, and **the successful poll that
the action itself kicked off cleared that same field** about 100 ms later — invisible. The
periodic poll (5 s) would have erased it shortly after anyway. Both the README and the code
comment promised that failures are reported where the user acts; they never were.

**How to apply:**

- **Split the channels.** "What the poll found" and "what the action answered" are separate
  fields, and a poll may only touch its own. The action's word wins when both hold
  something — it answers the click the user just made.
- Make it a rule that the action's word is cleared only by **the next action or by closing
  the panel**. "A successful retry clears the failure it retried" then follows for free.
- **Extract the rule into a pure value type and test it.** "An action's message survives the
  successful poll that follows it" is a few lines of test; the UI layer cannot catch that
  regression.
- Verify on a real machine by **provoking the error and sampling the view's elements for
  several seconds**. A single screenshot cannot tell "shown for 100 ms" from "never shown".

### A menu-bar app's popover can be verified from a script (AX tree + CGEvent)

**Symptom:** treating "the popover's content cannot be checked programmatically" as a given
puts regression detection on human eyes — and a defect like **a banner shown for 100 ms**
is invisible to them too. (task-clock-gui, 2026-09)

**Why:** what fails is `entire contents`, not the accessibility tree. Descending from
`UI element 1 of window 1` **by hand** enumerates the SwiftUI content with role / value /
position / size, and displayed text appears in `value` — so assertions about the screen can
be machine-checked.

**How to apply:**

- Assert on text and state by walking the AX tree (`entire contents` can come back empty).
  "Act, then sample the tree for several seconds" also measures **how long** something stays
  on screen.
- **Synthesize clicks with CGEvent.** System Events' `click at` drives AppKit buttons but
  may not fire SwiftUI's `.onTapGesture`. Post mouseMoved → leftMouseDown → leftMouseUp
  50-80 ms apart from a small helper binary.
- **An AX position is the element's top-left.** A 10x11 icon button must be clicked at its
  centre (+5,+5); the exact corner misses.
- **Re-read the tree immediately before every click.** One extra banner shifts every row
  below it, and reused coordinates hit **an unrelated control** (in our case it manually ran
  a different task). With the panel closed, those coordinates land in whatever app is behind.
- Since the clicks have real side effects, add **a disposable target to the config** for the
  test and remove it afterwards — never rehearse on production tasks or switches.

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

**How to apply:** Scaffold `build-darwin-arm64` /
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

## What an app looks like is decided by the SDK it recorded linking against — and a toolchain update rewrites that silently

**What happened:** A SwiftUI menu-bar app re-released with only string and layout
changes came back drawing with the previous generation of window chrome (square
corners). The Info.plist was identical apart from the version string. The
difference was in the binary's `LC_BUILD_VERSION`:

| build | minos | sdk |
|---|---|---|
| previous release (looked right) | 14.0 | **26.5** |
| new release (square corners) | 14.0 | **14.0** |
| other Swift GUI apps on the same machine, built before the update | 13/14.0 | 26.5 |

macOS reads **which SDK an app was linked against** to decide which design
generation to render it with. The newer toolchain (Xcode 27 / Swift 6.4) stamps
that field with the *deployment target*, so the app declared an old SDK and got
the old look.

**Why nothing caught it:** the build, the signature, the notarization and the
whole test suite pass either way. Only the OS's rendering changes, and no
artifact check looks at that. The symptom also appears **only when the deployment
target is older than the current SDK**, so an app targeting the current OS does
not reproduce it on the same toolchain — a sibling app looking fine is actively
misleading.

**How to apply:**
- Pass the SDK version to the link step explicitly in release builds:
  `swift build -c release -Xlinker -platform_version -Xlinker macos -Xlinker <min> -Xlinker <sdk>`.
  Setting `SDKROOT`, or passing `-Xlinker -sdk_version` on its own, does not work
  (both measured). Derive `<min>` from `Package.swift`'s `.macOS(.vNN)` so the
  deployment target is stated once.
- **Add the check to `verify-release`**: read `LC_BUILD_VERSION`'s `sdk` with
  `otool -l` and fail unless it equals the current SDK. A visual regression can
  only be judged by a human, so this is the one proxy a machine can gate on.
- After updating a toolchain, **diff the first release artifact against the
  previous one at the binary level**, not just the sources. `LC_BUILD_VERSION`,
  signing attributes and embedded versions all change without any source diff.
- **Count the sweep population from the workspace root, `_wip/` included.** The
  fix was rolled out to every Swift GUI in the series checkouts on 2026-09-17; a
  project scaffolded under `_wip/` the same day was not in that enumeration and
  produced a deployment-target-stamped build the next morning (spice-client,
  2026-09-18). `find <workspace> -name Package.swift` is the population, not
  `find <series>`.

## When every view is correct and the drawing is not, the thing to dump is the layer tree

**Symptom:** The top 44pt of a session window was a black band, and the status text,
checkbox and two buttons that live there were not visible at all. Accessibility
reported all four controls present at the right positions and sizes. A recursive dump
of the view hierarchy showed every view with `isHidden` false, `alphaValue` 1.0 and
the intended frames. The window background was the system colour and the appearance
was light. **Nothing was hidden and nothing was transparent, and nothing was visible.**

Reading the code found nothing. A minimal reproduction harness — the same structure, a
control row above a drawing view inside an `NSHostingView` — was tried five ways and
drew correctly every time.

**Why:** The fault was in the layers, not the views. Dumping the layer tree showed one
content layer inside the drawing view measuring 1024x752 where its view was 1024x676,
and the parent layer's `masksToBounds` was false, so the extra 76pt was not clipped
and painted straight over the control row above it. **The view hierarchy and the layer
hierarchy are different things, and the second is under no obligation to respect the
first's bounds.** Checking view frames can never see this.

**How to apply:**

- For "laid out but not visible", **dump the layer tree**, not the view tree: each
  layer's `frame`, `opacity`, `isHidden`, `isOpaque`, `backgroundColor`, whether
  `contents` is set, and `masksToBounds`. One layer whose frame disagrees with its
  view's bounds is the answer.
- Add the diagnostic behind a temporary environment variable and remove it afterwards.
  Attaching to the running process is not an option for a signed app under the
  Hardened Runtime, because the debugger cannot attach.
- When you embed someone else's drawing view — a dependency's — in your layout,
  **clip it to your own frame**. Clipping is not cosmetic here; it is the structural
  guarantee that whatever they do stays inside the area you gave them. `.clipped()` in
  SwiftUI, `masksToBounds` on the parent in AppKit.
- A reproduction harness is good at establishing what is *not* the cause, and that is
  all it does. **Elimination and observation are different tools**; switch to observing
  the real thing sooner than feels natural.

## A menu bar panel's own state is the only state you can read at click time

**Symptom:** clicking a menu bar item again, right after its panel closed, did nothing — and
clicking repeatedly kept it shut. Reported from use, on three separate apps that shared the
same shape.

**Measured (macOS 27.0, 2026-09-21, synthetic HID clicks with every event logged):**

- `NSPopover.isShown` stays **true for about half a second** after `performClose` *and* after
  `close()`, until `popoverDidClose`. Deciding from it turns a re-click inside that half
  second into another close: on the release builds the re-click opened the panel **0 times out
  of 10** at every gap tried, from 60 ms to 350 ms.
- `popoverDidClose` for one close arrives **after a show that followed it**, so the delegate
  callback cannot be believed on its own either.
- The panel window's `isVisible` flips false within milliseconds of a close — but it is also
  false between a `show` and the moment the panel appears, because AppKit queues a show that
  starts during a close animation behind it (about 0.4 s).
- One click on the item produces **two events, either of which can be missing**: a global
  mouse-down monitor sees it first, the button's action arrives 23–41 ms later, and of eight
  well-separated clicks eight were monitored and five produced an action — the missing ones
  being clicks that closed the panel.
- A panel **shown from the monitor** (on the mouse-down or the mouse-up) is dismissed by AppKit
  inside the same click, every time. Only the button's action can open it.

**How to apply:**
- Keep the fact in your own value — "is the panel up" — set when you show and when you close,
  and read nothing of AppKit's to decide. Reconcile with `popoverDidClose` by counting the
  closes you asked for and consuming their late reports; a report you did not cause is the one
  that carries news (a `.transient` dismissal, the Escape key).
- Give the monitor the dismissing and the action the opening. Suppress an action that arrives
  within ~0.1 s of the monitor closing the panel for a click on the item: it is that click's
  second event. **Do not pair the two events by order** — with one of them missing, "the
  action of the click that just closed the panel" and "the action of the click meant to open
  it" are the same event, and an ordering rule swallows clicks.
- Expect a residual and measure it rather than claiming none: at a 60–100 ms gap the panel
  ended up closed once in ten, because the dismissed click's action can arrive after the
  window. Two clicks that fast are one gesture; a longer window brings back the defect.
- Verify at the layer the user touches. A synthetic-click harness is a few dozen lines —
  `CGEvent` posted at `.cghidEventTap`, the item's frame printed by the app itself, panel
  visibility from `CGWindowList` with `.optionOnScreenOnly` — and it reproduced the defect
  deterministically on the release build, which is what made the fix checkable. Pure tests of
  the decision state machine pass either way: they observe a layer below the defect.
- Fix every sibling in the same change. The third app had the same defect with a different
  mechanism, and nobody had reported it.

## At write time, neither CFPreferences nor the plist says whether a preference was saved

**Corrected twice on 2026-09-21.** This entry first prescribed a two-state record and a
process that stops "after `CFPreferencesSynchronize` fails"; then, after one measurement, said
to verify a write against the plist. Both were wrong, each for the same reason: designed on
something that had not been measured *on the real target*.

**Measured (macOS 27.0):**
- *An unsaved write* (a throwaway ByHost domain whose plist was made immutable):
  `CFPreferencesSetMultiple` + `CFPreferencesSynchronize` returned **true**.
  `CFPreferencesCopyValue` then returned the value that had been asked for — in the writing
  process *and in new processes* — for between 15 s and a minute, and afterwards the old
  value, with no error at any point. The plist held the old value throughout.
- *A saved write, throwaway domain:* the plist showed the new value 0 ms after `Synchronize`.
- *A saved write, the real global ByHost domain*
  (`~/Library/Preferences/ByHost/.GlobalPreferences.<host uuid>.plist`): the plist showed it
  **4–8 s later**; the API showed it at once. The throwaway result did not transfer.

So right after a write, the API cannot show an unsaved change and the plist cannot show a
saved one. They agree again within seconds (success) or about a minute (failure).

**How to apply:**
- A read-back through the API detects a value the OS *rejects or ignores*. It does not detect a
  failure to *save*. Say so in the product's limits instead of building on it; the cause is a
  Mac that cannot save preferences for any app. If it must be closed, the only parameter-free
  rule is "hold the change unconfirmed until the API and the plist agree", which costs every
  change a wait of seconds.
- What needs none of this: after a *reported* failure, put the way-back record back as it was
  **without reading** (the read can return the value that was asked for); after a write whose
  read-back equals the state before it, leave the record exactly as it was, so a value someone
  else set is never adopted as the app's own.
- **Measure the failure before designing for it, on the real target.** Three designs were
  written and blocked in review, all assuming the failure is *reported*; a twenty-line probe
  showed it is not. The next design trusted a probe of a throwaway domain, and the hardware
  test on the real domain failed every successful write. A stand-in measures the stand-in.
  When a measurement must touch real user state, do it under a guard that sets the user's
  values aside, restores them on every exit path, and verifies the restore.
- When reviews keep finding the same class of defect in successive fixes, stop patching and
  test the premise.
