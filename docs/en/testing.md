# Testing

Test design, and techniques for catching pre-release the defects unit tests
cannot reach. Each entry follows **symptom → why → how to apply**.

---

## All-green unit tests still need real-data E2E and real-binary simulation

**Symptom (three kinds of real cases):**
- A mail-analysis tool with 47 passing unit tests revealed 6 bugs in an E2E over
  12 real e-mails (false negatives from unlisted legitimate cloud URLs,
  subdomain-matching gaps, broken Base64 header decoding, auth-only false
  positives…).
- Real-binary simulation surfaced missing-data-file errors, flag-less run
  failures, and missing dates in output. Output-format and UX problems are
  invisible to unit tests in principle.
- An MCP server with green units + a fully passing scripted E2E exposed **three
  defects the moment it was used as a real MCP client**, and subsequent releases
  kept being driven by "defects that only appeared in use". A scripted E2E only
  walks the steps its author imagined.

**How to apply:**
- Keep a gitignored `testdata/samples/` with real data and run E2E before
  releases.
- Manually walk the built binary through basics: happy path → no-config start →
  empty state → eyeball the output format.
- **Connect MCP/tools to your own client and take a lap before publishing** —
  improvised operations, not a script (stepping outside the imagined path is the
  point). Third-party testing is even better.
- Never trust GUI error display until an error is deliberately triggered on the
  real app.

## A pipeline that ends in delivery is not verified until something is delivered

**Symptom:** A feed-digest tool passed 315 unit tests and a real-data run from
collection through to a rendered file. Every defect below survived all of that
and appeared only when a message was actually sent.

- Inline `[title](url)` links reached the destination as **titles with no
  address**. Its markup converter had no inline links and dropped the URL
  rather than translating it — so a tool that selects what is worth reading
  delivered everything except the way to read it.
- The character limit was measured before a `*(continued)*` prefix was added to
  every part after the first. The real message came out **seven characters
  over**, and the destination rejects rather than truncates.
- A run producing two parts after one that produced four left `msg-03` and
  `msg-04` on disk. The sender walks the parts in order, so two current
  messages would have been followed by two from the previous run.

None of these is reachable from a fixture. The first needs the destination's
own converter, the second needs multibyte text at real length (a Japanese
digest is ~3 bytes to the character, so a byte-based limit is silently wrong in
both directions), and the third needs the transition from more parts to fewer.

**How to apply:**
- Treat "rendered correctly" and "arrived correctly" as different claims, and
  make the second one before release. Read the message *back* from the
  destination — what is stored there is what the reader sees, and it is not
  always what was sent.
- Turn each defect into an invariant over the pipeline's own output rather than
  a case: *every item's URL appears in the delivered text, for every
  destination flavour*; *every part is within the limit, marker included*. An
  invariant keeps holding when a new flavour or a new marker is added.
- Clear previous output before writing a variable number of files. Anything
  that consumes them by enumeration will otherwise consume the leftovers.

## A drill runbook is unfinished until it has been run once

**Symptom:** A seven-step monthly drill runbook for a fallback tool was
written, and its first run **rewrote three of its own steps**. (1) The
read-only step meant to exercise the tool path asked what the project does
and what its build commands are — the agent injects the project's
instruction files into its system prompt, so a correct answer came back with
zero tool calls. Correct answer, path never exercised. (2) Auto-approve was
on by config, so the step meant to exercise the approval gate would have run
unattended. (3) The sandbox-containment step asked the model to write outside
the project; the model read the restriction and declined to try — a pass that
tests nothing — while an earlier run of the same request did try and hit the
gate first.

**How to apply:**
- **Run it once after writing it.** Steps changing is the normal outcome; if
  nothing changed, doubt that it was really run. When you edit a runbook, run
  at least the step you changed — write that rule into AGENTS.md so it sticks.
- **Suspect three shapes of "looks like a pass, verifies nothing":** a
  question answerable without traversing the path under test (the answer is
  cached somewhere else); a check that depends on the subject's cooperation
  (replace it with a direct path); a check whose result varies between runs
  (that is not a check).
- **Pin state-dependent defaults at the top of the procedure.** "It happened
  to be on" is enough to hollow out a check.
- Alongside the checklist, include a step that **completes one real task with
  the tool alone**. A checklist shows the thing runs; only real work shows it
  is usable.
- **Make the verdict binary: pass or issue.** Allowing "mostly worked" makes
  the record meaningless. Record a skip as a skip — an unrecorded gap reads as
  a pass.

## A feature that never fires still passes every unit test

**Symptom:** Automatic conversation-history compaction was implemented. Unit
tests all green. In real use it **never fired once, and said nothing**. Two
causes had stacked. (1) The threshold check needed a context-window value
resolved asynchronously at startup, but only the interactive path started that
resolution — the one-shot path left it at 0, so the check was always false.
(2) The cut-point invariant was "only at a user-message boundary", and a long
agent loop contains exactly one user message, at the very beginning — so the rule
**structurally excluded the case the feature exists for**.

**How to apply:**
- **Unit tests supply the condition directly, so they never verify that the
  condition arises in production.** For anything threshold- or condition-gated,
  run the real binary with the threshold swung to an extreme value and **watch it
  fire** before calling it implemented.
- For a feature depending on an asynchronously resolved value, count the wiring
  per entry path (interactive / one-shot / scripted / resumed). Distrust any
  design where an unset value silently no-ops; make it say something instead.
- Keep one test that applies the invariant to the most typical usage. Reading the
  rule will not reveal that a safety-motivated invariant excludes the main case.
- If "cannot fire" is reachable, report it. **A silent no-op is
  indistinguishable from a missing feature.**

## Design external dependencies mockable from the start

**Symptom:** A test launched a real browser. It was later retrofitted into an
injectable function variable — it should have been designed that way from the
start.

**How to apply:** Inject external dependencies (browser launch, external HTTP,
clock) via package variables or interfaces; swap them in tests.

## xattr write failures can be injected for real with a deny ACL

**Symptom:** Wanting to test the path where `setxattr` fails (volumes that can't
take quarantine). Mock seams add mutable globals to production code and skip the
real syscall path; disk images are slow and environment-dependent.

**How to apply:** An inheritable deny ACL on the destination directory makes
xattr writes fail with EPERM even on files you own:

```sh
chmod +a "$(id -un) deny writeextattr,file_inherit,directory_inherit" <dir>
```

Newly created files inherit the ACL, so everything under the directory fails
xattr writes — the same shape as extracting onto an xattr-less volume. Fast
(0.13 s measured) and exercises the genuine failure path. Run chmod from the
test and skip if it fails (insurance against environment differences).

## Automate MCP-server E2E with a dummy JSON-RPC client

**Symptom:** Tests with an LLM in the loop are nondeterministic, slow, and cost
tokens. Yet protocol round-trips / error paths / timeouts / workspace isolation
are all verifiable with fixed JSON-RPC.

**How to apply:**
- Implement a harness that spawns the server as a child process and speaks
  JSON-RPC over stdin/stdout (`Start / Call / Notify / CallTool / Close`).
- Isolate behind a `//go:build e2e` tag; gate container-requiring tests with an
  env var.
- Minimum scenarios: lifecycle / errors / timeout / sequential workspace
  isolation.
- Real-LLM verification (Claude Desktop etc.) can wait until the end — but the
  separate "use it as a client yourself" lap is still required (see above).
- Native-library stdout contamination does not reproduce in stub builds — E2E
  against the real engine (see mcp-server-design.md).

## Green tests prove nothing when the expectation itself is wrong

**Symptom:** code mapping an external API's status values onto UI behaviour had
unit tests, all passing. The feature still shipped in a state where nobody could
use it, and only a person touching the real build found out.

**Why:** the tests faithfully verified *the mapping I believed was correct*. The
mapping was wrong, so the tests only pinned the mistake in place. What a given
status value from an external API actually means is **an assumption, and an
assumption cannot be tested against itself**.

**How to apply:**
- When mapping an external API's states onto UI behaviour, do not treat those
  tests as confirmation of the specification. A misread doc passes them.
- Be especially wary of branches that **remove a capability** ("disable the
  control in this state"): get one wrong and the whole feature becomes
  unreachable. List them explicitly as things to check on the real build.
- Enumerate what only a real device can settle **while planning**, and say so.
  Discovering it later is worse, because a wall of green tests will have been
  informing the judgement in the meantime.

## Verify a GUI on the assumption that what you can see and what actually runs are independent

**Symptom:** a menu-bar app's bar label rendered correctly, which was taken as
evidence the app worked. The panel opened by clicking it had in fact been empty
since the first version.

**Why:** the bar label and the panel are separate view hierarchies; either can be
broken while the other is fine. The panel is also awkward to open
programmatically (accessibility prompts, clicks landing on the frontmost
process), so it falls out of automated checking easily.

**How to apply:**
- "The label is showing" is not a verification. **Count each surface of the UI as
  its own thing to check.**
- Where a surface cannot be opened automatically, **write into the plan that a
  person has to look at it**. Left vague, an unchecked surface gets treated as
  checked.
- Behaviour that only appears over hours — a frozen timer, a counter that should
  reset at midnight — is verified from the stored data after leaving it running.

## Driving a menu-bar (LSUIElement) app for E2E: CGEvent + CGWindowList + per-window screencapture

**Symptom:** a menu-bar-resident app's panel could not be opened from a test
script, so its states went unverified (see the entry above). AppleScript does
not help: `System Events`' `key code` never reaches a Carbon
`RegisterEventHotKey` handler, and an `NSStatusItem` action does not fire from
an AppleScript click.

**Why:** the panel has no scriptable surface. What it does have is a global
hotkey and a window, and both are reachable at a lower level.

**How to apply:** three pieces, all from a small Swift helper the harness
compiles once.

- **Open it** — post the hotkey as a `CGEvent`
  (`CGEvent(keyboardEventSource:virtualKey:keyDown:)` →
  `post(tap: .cghidEventTap)`). Requires the posting process to be
  Accessibility-trusted (`AXIsProcessTrusted()`).
- **Confirm it opened** — `CGWindowListCopyWindowInfo([.optionAll], kCGNullWindowID)`,
  filtered by owner name; read `kCGWindowIsOnscreen` and `kCGWindowNumber`.
- **Look at it** — `screencapture -x -o -l <windowid>` captures that window
  alone. Prefer this over a full-screen capture: it keeps unrelated windows —
  other people's chat, mail, tickets — out of the image entirely.

Read the hotkey out of the app's own `UserDefaults` before sending it, rather
than assuming the shipped default. The stored value is typically
`NSEvent.ModifierFlags` (shift `0x20000`, command `0x100000`, option `0x80000`),
**not** a Carbon mask — sending the wrong chord looks exactly like a broken
hotkey and invites a false bug report.

**The failure mode to plan for:** synthetic keystrokes go to whatever holds key
status, which is frequently *not* the app under test — a non-activating panel
takes key status inconsistently, and macOS focus-stealing prevention can deny
activation for ~30 s after launch. Every keystroke that misses lands in the
user's frontmost application, typing stray text into it and setting off alert
beeps. So: verify the window has key status before typing, keep the typed
strings short and inert, and **stop the technique the moment it stops landing**
instead of retrying — the retries are what reach the user's other windows. This
harness is well suited to opening a window and photographing states; it is not a
reliable text-input driver.

**Complementary, not a replacement:** the states this cannot reach (an IME
composition, a first-time model download) still need unit tests over pure
functions plus a person looking once.

---

## Check layout by rendering off-screen from the test target, not by launching the app

**Symptom:** unit tests over pure functions touch strings and arithmetic only, so
whether a string actually fits the real width is undetectable by construction.
Adding a single display line can push the longest branch into truncation, and no
amount of reasoning settles it. Driving the real app with synthetic events (see
the previous entry) does settle it, but it is too heavy — and too leaky — to reach
for on every layout change.

**Why:** a SwiftPM executable target is importable from tests with
`@testable import`. Put the view in an `NSHostingView` and its layout resolves
without launching the app; `bitmapImageRepForCachingDisplay(in:)` plus
`cacheDisplay(in:to:)` writes it to a PNG. The same layout engine and the same
real fonts run, so wrapping, truncation and true glyph widths all show up.
Nothing appears on the user's screen.

**How to apply:**

- Put a throwaway renderer in the test target: build
  `NSHostingView(rootView: TheView().environmentObject(model))`, give it **the
  production width** (e.g. the popover's content width plus padding), and call
  `layoutSubtreeIfNeeded()`.
- Take the height from `fittingSize.height` rather than pinning it. You are
  looking for content that does not fit; a fixed height hides the evidence.
- Build state by assigning to the `@Published` properties directly. This verifies
  rendering, so let it call no subprocess and no network.
- **Render one image per branch and look at all of them** (normal / warning /
  over / no-data). What breaks is usually the branch carrying the longest string,
  and that is not the everyday branch.
- Delete the renderer once you have looked. Keeping snapshots as regression tests
  buys failures from OS font and system-colour changes; keep them only as a
  deliberate choice.
- Save the synthetic-event route for what only a real app does: permission
  prompts, actual placement in the menu bar, behaviour right after launch.

---

## When two defences cover one failure, an observation explained by either is evidence for neither

**What happened:** An MCP server guarded stdout in two layers — (1) replacing the
runtime's log callbacks at the source, and (2) duplicating the real stdout for the
transport and pointing fd 1 at stderr. A real end-to-end run showed stdout carrying
only JSON and stderr carrying nothing, and that **nearly went down as proof both
layers worked**.

**Why:** stderr was empty because layer (1) filters info-level chatter, so **layer (1)
alone explains the entire observation**. It says nothing about whether (2) ever ran.
A broken (2) produces exactly the same result.

**How to apply:** When you build defence in depth, **verify each layer
independently**. Here that meant a unit test driving the mechanism directly:

```go
protocol, _ := claimStdout()
protocol.Write([]byte("PROTOCOL\n"))   // must reach the transport
os.Stdout.Write([]byte("STRAY\n"))     // must reach stderr
// assert the protocol stream never sees "STRAY"
```

The general rule: **if observation O is explained by defence A or by defence B, then
O is evidence for neither**. A green integration test does not prove each layer is
alive.

---

## Verify the fixture before blaming the code

**What happened:** Validating speaker diarization meant synthesising a conversation
from two voices, `say -v Kyoko` and `say -v Otoya`. Half an hour went into
suspecting the implementation for reporting one speaker where there should have been
two. The cause was that **`Otoya` was not installed, and `say` fell back to the
default voice without a word**. All four clips were the same speaker, so reporting
one speaker had been correct all along.

**Why:** Generative commands often answer a missing resource with a **plausible
substitute** rather than an error — `say` voices, font selections, model names,
locale identifiers. The output file is produced normally, so nothing hints that the
fixture is wrong.

**How to apply:** Confirm a synthesised fixture actually has the property you need,
by some means other than the thing under test. At minimum, check the named resource
exists first (`say -v '?'`, a font list, a model list). Reading a correct answer as a
bug costs more than the bug would have.

## "Not found" is only an answer when every source actually answered

**What happened:** A lookup tool queries OTX under two indicator types, because
upstream indexes a name under exactly one of `domain` and `hostname` and answers
`200` either way. During a live run one type returned 429 while the other returned
zero pulses, and the tool printed **"no community report names this indicator"
and exited 0** — a clean bill of health manufactured out of a transient error.
Every stubbed test passed; the defect only appeared against the real API.

**Why:** "Nothing matched" and "we could not ask" are **identical in the data**
and opposite in meaning. Any code that folds a failed source into an empty result
set will report the second as the first, and a negative answer is the one nobody
double-checks — a false positive gets scrutinised by whoever acts on it, while a
false negative is filed and forgotten. Mock-based tests cannot catch this, because
the failure only exists when one source fails and another succeeds.

**How to apply:**

- Carry an explicit `incomplete` flag alongside the results, and treat
  "empty **and** incomplete" as a distinct state with its own name — not as a
  boolean that a caller has to remember to check twice.
- Say it in the output. A user-facing `INCONCLUSIVE` beats a silent zero.
- Make the exit code carry it. Scripts read exit 0 as "clean"; an unverified
  negative must not exit 0.
- Name the source that failed. "Something went wrong" leaves the reader unable to
  judge how much of the answer to trust.
- When an upstream exposes several endpoints for the same logical object and
  **all of them return 200**, choosing wrong is not an error — it is a silent
  empty result. Query the alternatives and report which one answered.

## Never state a metric of a jittery system from a single run

**What happened:** Comparing two models on a local inference engine, one was
reported as falling into a run of 48 identical consecutive segments over a
39-minute recording. Re-run later with the same binary, the same model, the same
audio and the same settings, the longest run was 19. Three runs per model showed
what one run could not: **the volume of output is stable** (character count
varies by ±1%) while **the structure of the failure is not** (longest run 19 /
45 / 48). Tiny floating-point differences from multi-threaded reduction order
flip decisions at probabilistic branch points.

**Why:** "It is deterministic, so one run is enough" stops holding once the
numerics are parallelised — and it fails unevenly. Aggregate quantities stay
put while the discrete decisions near a threshold (enter a loop, split a
speaker, fall back a temperature) move. **If the first run was a tail of the
distribution, the number you published does not reproduce, and the person who
sees that has reason to doubt every other measurement you took.**

**How to apply:**

- **Establish whether the system jitters before you start comparing.** Run the
  same input twice. If the outputs differ, every metric from then on is a range
  over n≥3, not a value.
- **Report ranges, not points.** "19–48 across three runs", not "48".
- **When setting a threshold, justify it by two non-overlapping ranges.** In the
  example, looping runs were 19–48 and non-looping ones 2–3, so a threshold of 6
  sits in the gap. A threshold placed next to a single observation misfires on
  the next run.
- **Do not treat stable quantity as evidence of stable structure.** Character
  counts agreed to within ±1% and said nothing whatever about whether the loop
  would reproduce.

## Output with no ground truth can still be measured structurally

**What happened:** Two transcription models needed comparing on a 39-minute real
recording, but no reference transcript existed, so error rate was unmeasurable.
Metrics that need no ground truth were measured instead — segment coverage, runs
of identical consecutive text, the largest gap between segments, the skew of
speaker labels. That was enough to quantify the differences that decided the
choice: one model emitted 1.45× the text, and only one fell into repetition
loops.

**Why:** Data without ground truth is usually written off as unmeasurable, but
**broken output has a shape**. Repetition loops, dropped audio and over-splitting
are detectable from structure alone, without reading a word of content. And
those failures characteristically arrive **perfectly well-formed** — valid JSON,
every field populated, no error raised — so nothing notices them unless
something measures the structure.

**How to apply:**

- Separate metrics that need ground truth (accuracy) from those that do not
  (structure), and compare on the latter first. It often settles the choice on
  its own.
- **Only conclude from non-overlapping differences.** Structural metrics are
  proxies; a narrow margin in one means nothing.
- **Write the metric scripts to emit no content at all.** Counts and rates only,
  and the same script can be pointed at confidential or personal data. Real data
  without a reference transcript is usually unreferenced *because* it cannot
  leave the building — the constraint and the missing ground truth share a
  cause.
- Watch for metrics that are **ambiguous alone**. High coverage may mean less
  audio was dropped, or it may mean the model hallucinated over silence and
  music. Disambiguate with an independent third observation (here, a VAD).

## Reconcile what an enumerator returned against what you meant to measure

**Symptom:** While investigating whether SMART data could be read from a
USB-attached external SSD, a diagnostic tool's auto-enumeration mode (`--scan`)
was run, a command was issued against the device it returned, and the resulting
I/O error was reported as evidence that *this SSD* cannot be read. In fact the
enumeration had **not listed either of the two drives under investigation**; it
returned exactly one unrelated device — an **empty drive caddy** hanging off a
different hub. That error is equally explained by "no media present", so it was
no evidence at all for the constraint being investigated. Re-measuring by naming
the two real drives by device path produced a different error (the conclusion
itself did not change).

**Why:** An enumerator returns *what it found*, not *what you pointed at*. A
single-result listing invites the assumption that the result is your target.
Worse, here the **conclusion happened to be correct** — "not readable over USB"
was independently supported — so the fact that the *evidence* was wrong never
surfaced. A correct conclusion propped up by faulty evidence gives no way to
tell what broke when a premise changes (different hardware, a later OS release).

**How to apply:**

- Reconcile enumerator output against your target by an **independent
  identifier** (device path, serial, capacity, logical name). Do not trust the
  label the tool attached on its own.
- **Compare the number of items enumerated against the number you know is
  attached.** Two drives connected but one result means the enumeration itself
  is suspect. A non-zero count that does not match is more dangerous than zero.
- If a path exists that names the target explicitly, re-measure through it.
  **Auto-enumeration is a discovery tool, not a verification tool.**
- "The conclusion is right, so the evidence must be" does not hold. Audit the
  conclusion and the evidence separately.

## Do not infer physical layout from a position in a logical tree

**Symptom:** In the same investigation, the target SSD appeared under an
SoC-internal USB controller in the OS device tree. That was taken as proof it
was plugged into a USB-only port, and the operator was told to move it to a
Thunderbolt port. **It had been in a Thunderbolt port the whole time.** On that
platform, USB traffic from a Thunderbolt port surfaces under the same USB
controller, so port type simply cannot be derived from tree position. The
operator overturned it in one sentence: "no, that's where it already is."

**Why:** An abstraction layer's tree describes *how a device is handled*, not
*where it is attached*. And on systems where one physical port surfaces
separately in several planes (here a USB plane and a Thunderbolt plane), looking
at only one plane makes the device look absent from it.

**How to apply:**

- Calibrate physical layout against a **device whose location is already
  certain**. Here, "a Thunderbolt hub only fits a Thunderbolt port" plus "that
  hub appears under the USB controller in question" settled it in one step.
  **Finding one known point beats stacking inferences.**
- **"No device connected" in one plane is not evidence of absence.** Inverted,
  the asymmetry is a strong diagnostic signal — the Thunderbolt bus insisting
  nothing was attached was precisely the evidence that the device was a pure USB
  bridge establishing no PCIe link.
- **Do not infer from tool output a physical fact a nearby human can answer in
  seconds.** Asking is faster, more reliable, and cheaper to be wrong about.
  Infer only when there is nobody to ask.

### Verify concurrency concerns with a concurrent test, not by reading — the bug lives next door

**Symptom:** asked whether timestamp-based session ids collide under parallel
execution, code-reading found the suspected layer safe (O_EXCL + suffix retry
+ flock). The 16-way concurrent test written to PROVE that safety caught a
real race in the ADJACENT layer (the state-dir ownership marker) at ~50%
frequency. A reading-based "it's safe" answer would have shipped the bug.

**How to apply:**
- On any concurrency doubt, write the concurrent regression test first
  (start-channel for simultaneous release, `-race`, several rounds) whatever
  you expect the verdict to be — the test exercises the whole path, not just
  the suspected hypothesis, so adjacent defects surface.
- Confirm goroutine findings with real simultaneous processes (O_EXCL, flock,
  rename are kernel vocabulary, so the semantics match — but the E2E catches
  wiring mistakes).
- Measure before answering "no issue": the negative-answer integrity duty
  applies to concurrency too.

### The single production call site is the weak point of injected features — pin the wiring with an AST test

**Symptom:** a UI language-catalog feature shipped with all unit tests green
and a non-interactive E2E pass — while the one production TUI-constructor
call omitted the catalog argument, so the whole TUI silently ran on the
English fallback for a full release. Unit tests inject the dependency
themselves and cannot see a wiring gap; the E2E only exercised the surface
that WAS wired.

**How to apply:**
- Third pattern of "implemented but silently never fires": **an options-struct
  field with a nil default degrades quietly** — the graceful fallback hides
  the missing wire precisely because it is graceful.
- Behavioral tests cannot reach a literal inside a monolithic init function.
  **Parse the production file with go/ast and assert the constructor literal
  sets the required fields** — a few dozen lines, robust to formatting, and
  it fails the moment the field disappears.
- Run one E2E per surface the feature passes through, not per surface where
  it visibly worked: here only the non-interactive (cmd) side was measured,
  and the interactive (TUI) side was the broken one.
