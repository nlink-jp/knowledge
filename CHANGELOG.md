# Changelog

## 2026-08-15

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
