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
  pane that undoes it — never show an on switch that delivers nothing.
- The same holds for location, calendar and other OS permissions.

### A change in a second ObservableObject does not reach views that do not observe it

**Symptom:** a settings checkbox updates instantly while the **menu-bar display
stays stale until the next periodic refresh**.

**Why:** when the displayed values (a model) and the choice of how to display
them (a preferences object) are separate `ObservableObject`s, changing the latter
does not invalidate a view observing only the former.

**How to apply:** have the view hold **both** of the things it actually depends on
as `@ObservedObject`. Places outside the injected environment — a `MenuBarExtra`
label, for instance — are the easiest to miss.

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
