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

## Mocks exercise the error path; nobody reads the sentence it produces

Mocks cover the **branches** of a failure path. What they cannot cover is
whether the sentence assembled at the end of that path **means anything to the
person who reads it**. A substring assertion goes green on wording that is
actively misleading.

**Case (2026-08-23, mcp-bridge v0.1.0 pre-release verification).** Connecting to
the live Slack MCP server with a revoked token surfaced two defects that no
unit test had failed on.

1. **The cause vanished along the path.** A 401 invalidated the credential, the
   retry then failed for lack of one, and the only text reaching the user was
   `no access token`. True, but it is the *after-effect* of invalidation rather
   than the cause, and the reader goes looking for an empty token file. Fixed by
   carrying all three facts to the end: the 401, the reason no replacement
   exists, and the command that resolves it.
2. **A successful run printed an error line.** With a self-signed loopback
   callback the browser's first connection always fails the handshake — that
   failure is exactly what produces the warning the user clicks through — and
   `http.Server`'s default ErrorLog printed it as `tls: bad certificate` on
   every login, right after the tool's own note saying the warning was expected.
   **A successful login that reads as a failure.**

The mocks took the same branches in both cases. The only difference was running
it and **reading the output as a human**.

**How to apply:**

- In pre-release real-data verification, deliberately trigger the **failure**
  paths and read the terminal output as written, not just the happy path. One
  invalid credential is usually enough.
- Do not test error text with substring matches alone: **pin the words that must
  not appear**, too. (For case 1, the test now asserts `no access token` is
  absent.)
- When a dependency carries **its own logger** (`http.Server.ErrorLog` and
  friends), run the production path once to see what it emits. Output you did
  not write does not show up in tests you did write.

Related: [[all-green-unit-tests-still-need-real-data-e2e]]

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

## Before calling something "measured", ask whether the probe input represents the real thing

**Symptom:** how a terminal Markdown renderer treats over-wide code-block lines
was "measured" with a single 132-cell line of box-drawing characters **containing
no spaces**, and the conclusion — "it neither truncates nor wraps; lines pass
through unmodified" — went into a design document as a measured fact. An
independent review re-measured with real box art, which **contains spaces between
boxes**, and found the renderer word-wraps code-block lines at spaces: any
drawing wider than the wrap width was sheared into interleaved fragments before
the terminal ever saw it. The probe input happened not to carry the mechanism
(word boundaries), so the behavior that mattered was unobservable.

**Why:** a synthesized minimal input can lack exactly the ingredient the property
under test operates on. Claiming a general statement ("this renderer does not
wrap") from an n=1 probe manufactures one false "measured" per missing
ingredient — and a claim wearing the "measured" badge is rarely re-verified,
while the designer, whose probe is optimized for their own hypothesis, cannot
see what it omits.

**How to apply:**
- Take probe inputs from **the real artifacts that flow through the path in
  production** (real renderer output, real model output, real logs). When you
  must synthesize, deliberately include the ingredients the property could
  operate on: spaces, double-width characters, control characters, length.
- When a document claims "measured", **record the input's provenance** (what was
  fed, and how). A measurement is only a claim when a re-measurer can judge its
  representativeness.
- Give any independent verification pass the instruction "re-verify every claim
  against code or a probe before accepting it" — that instruction is what caught
  this case. A sibling precedent: hand-written test expectations that lacked the
  real renderer's decorations shipped a false negative — expectations and probes
  alike are built from real artifacts.

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

**The status item itself cannot be observed through the window list (macOS 27.0, two
observations):** the check above looks at the **panel's window**, which the app opens.
Using the same method to confirm that the menu bar item is showing leads to a wrong
conclusion. Launching an app with one `NSStatusItem` left zero windows owned by that app's
pid, and the total number of windows in the list was the same before and after the launch —
no process gained a window. The item was visibly there. An empty result is not evidence that
nothing is shown. Confirm the item by eye, through the accessibility tree, or by having the
app report its own geometry. Also confirm that the probe can see the window server at all:
print the total window count alongside, so that "cannot see" is never mistaken for "does not
exist".

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
## Establish a log's timestamp semantics before drawing any conclusion about time

**Symptom:** A daily log-summary report announced "disk I/O errors yesterday". The
kernel lines (ATA command timeouts and link resets) were read correctly and a
drive-failure diagnosis was built on them — but the entries were from the **same
calendar date two years earlier**. `/var/log/messages` had gone 2.7 years without
rotation, and because RFC 3164 timestamps carry no year, the report's date match
picked up the identical date string from a previous year.

**Why:** The interpretation of the content was right. What was false was the
**time the evidence was assumed to carry**, and that invalidated the whole
diagnosis. Timestamp ambiguity has several independent layers — presence of a
year, presence of a timezone, re-rendering by the viewer, event time versus
ingestion time, conversion of relative time. None of them are visible while
reading the log; they all bite after the conclusion is drawn.

**How to apply:** Settle these before reading the content.

1. **Coverage** — `ls -la` (size), then `head -1` / `tail -1`. A quiet server
   whose log is tens of megabytes is already a signal.
2. **Does it carry a year?** — RFC 3164 (`Aug 22 06:37:29`) does not. **A
   year-less format breaks date matching the moment the file spans more than a
   year.**
3. **Timezone** — the writer's side (RFC 3164 has no offset either); the viewer's
   side (`journalctl` renders in the *reader's* timezone, so the same event shows
   a different clock time than the text log); and mixtures (containers on UTC,
   host on local time). **Where DST applies, one hour repeats and one hour does
   not exist once a year, breaking both string sorting and date matching.**
4. **Event time or ingestion time?** — forwarding and buffering turn it into the
   latter.
5. **Can monotonicity be assumed?** — "append order equals chronological order"
   breaks with multiple writers, NTP step adjustments, and merged files.
6. **Relative-time conversion** — `dmesg`'s `[12345.678]` is seconds since boot.
   `dmesg -T` derives absolute times from boot time plus the monotonic clock, so
   **it drifts whenever the clock is stepped**.

Do not settle a date from a log that carries only ambiguous time. Go to a source
that carries explicit time:

```
journalctl -k --since "YYYY-MM-DD" --until "YYYY-MM-DD" -o short-iso
```

journald stores realtime in UTC microseconds along with a boot ID, so year,
timezone and boot count are all unambiguous. The point is not "cross-check
against a second source" but **go to the source with better timestamp
semantics**.

To recover the year from a year-less log, look at where the date string lands:

```
grep -n "^Aug 22 " /var/log/messages | cut -d: -f1   # line numbers of matches
grep -n "^Jan  1 00:0" /var/log/messages             # year boundaries
```

The same date string appearing at widely separated line numbers proves the file
spans multiple years. Bracketing each cluster by the year boundaries fixes its
year (this relies on the monotonicity assumption in 5).

### A feature that learns from usage must be tested against usage-shaped data

**Symptom:** An approval-rule learner shipped with unit tests, an E2E,
and a threshold of "approved in three separate sessions". Its first
real session — 25 escalations, all approved — produced zero proposals.
The E2E had seeded exactly three sessions to satisfy the threshold, so
it measured whether the mechanism worked and never whether the
threshold was reachable.

**Why it matters:** Fixtures written to satisfy a threshold cannot
falsify it. The tests were green, the feature ran, and the defect was
invisible until real data arrived — the same class as a feature that
never fires, but hiding behind passing tests rather than behind silence.

**How to apply:**
1. **Shape at least one fixture like production, not like the
   threshold.** For a learner, that means the distribution real usage
   produces: work concentrated in one session, many one-off items, a
   long tail — whatever the domain actually looks like.
2. **Ask "is this bar reachable?" separately from "does this work?"**
   They are different questions and only the first needs real
   distributions. Answer it before shipping, with numbers.
3. **When the real report arrives, reproduce before diagnosing.** A
   synthetic transcript in the reported shape turned a plausible story
   into two measured causes in one test run — and became the
   regression test.
4. **Check whether the friction even has the shape you are counting.**
   Here half the problem was categorical: per-tool frequency cannot see
   friction made of many different tools called once each. No threshold
   over that counter is both safe and reachable, so the unit of the
   rule had to change, not its number.

## A constant that bounds a feature: measure the population its window covers

**Symptom:** An agent's risk evaluator gained a feature — the operator's
instruction rides along as alignment evidence — bounded, by intuition, to
**the first 3 rounds of a turn** ("early calls trace to the request; deep
calls serve sub-goals it never names"). Weeks later, reconstructing round
positions from every real session transcript: **70%** of model-tier
evaluations fell outside the window, and **63%** of turns placed their
*terminal* gated call — sends, saves, the actions whose alignment matters
most — outside it. Where records existed, every beyond-window escalation
had been hand-approved by the operator afterwards: in practice, false
alarms on read-only research calls.

**Why it matters:** A cutoff constant encodes the usage shape imagined at
design time. Real usage is heavy-tailed (rounds 8–48 held half the
volume), and **no constant fits a heavy tail** — sometimes the honest
measurement result is that no fitting number exists. This is a variant of
"a feature that never fires cannot be caught by precision — count the
denominator": **a window's coverage is invisible until you count its
denominator too.** The feature works, tests pass, and inside the window
it behaves correctly, which is exactly why nobody notices.

**How to apply:**
- When a feature is bounded by a cutoff constant (rounds, counts, sizes,
  elapsed time), measure after release **what fraction of the real
  population the window covers**. If coverage is a minority, question the
  cutoff itself rather than tuning the number.
- Log so the measurement is possible: include position (round number
  etc.) in decision events, or keep transcripts from which position can
  be reconstructed from the message sequence. This measurement used the
  latter.
- Whether the outside-the-window path produces false alarms is measured
  by pairing each escalation with the human's next decision. Escalations
  that are approved every time are friction, not defense.

## An external API's vocabulary and visibility vary by licence tier — measure with your own key, not the docs

**Symptom:** the filter vocabulary in the official documentation (hyphenated)
was rejected by the live API with 400, and the accepted vocabulary
(underscored) appeared nowhere in the docs. Even accepted types were
tier-dependent inside: content gated to higher licences answered **empty, not
an error**. The vendor's own published reference implementation used the
documented vocabulary and did not work as shipped (lookup CLI+MCP project,
2026-09).

**Why:** an API's answer is the product of endpoint specification × key
entitlement. Documentation tends to be written from the top licence's view,
and a reference implementation's existence is not evidence of correctness.
Empty, 400 and 403 all read as "unusable" but mean different things — without
separating them you conflate "does not exist" with "not visible from this
tier".

**How to apply:**
- Before integrating, run a sorting measurement **with your own key**: 400
  (vocabulary/syntax rejected) / 403 (endpoint refused) / empty (content
  tier-gated). Record the results with the date and the key's tier.
- If a privileges endpoint exists, read it first — one request beats
  inferring entitlements from behaviour.
- **Do not ship what you cannot exercise with your own key.** Speculatively
  implementing features that "should work" on a higher licence leaves
  permanently unverifiable code paths. Cut scope to the feature set you can
  verify, and state that decision and its reason in user-facing docs.
- Where a tier-gated empty answer reaches users as a tool result, the
  user-facing docs must say "empty ≠ nonexistent".

### golangci-lint truncates identical findings at three — list everything before a bulk fix

**Symptom:** errcheck reported "8" unchecked `defer x.Close()` calls; there
were 18. golangci-lint caps identical messages at `max-same-issues`
(default 3) and per-linter findings at `max-issues-per-linter` (default 50).
Each receiver name got three lines, so fixing what was shown and re-running
surfaced the "next" three — a whack-a-mole loop.

**How to apply:**
- Before a bulk fix, run `golangci-lint run --max-same-issues 0
  --max-issues-per-linter 0 ./...` to see the whole set, then fix it
  mechanically (a regex from `defer x.Close()` to
  `defer func() { _ = x.Close() }()`, for instance).
- One finding of a shape is a reason to grep every file for that shape.
  The displayed count is a floor, not the total.

### An E2E script aimed at a fixture must refuse to run in your own repository even with an empty argument

**Symptom:** An E2E script was generated through an unquoted heredoc
(`<<EOF`), so the `$1`/`$P` in its body expanded to empty strings at
generation time. The script ran `cd ""` (a no-op) and then
`printf ... > AGENTS.md`, replacing the repository's 700-line `AGENTS.md`
with the two-line fixture. The feature commit carried it; a review found it.

**Why:** "It points at the fixture" is the script's assumption, not a fact
the script checks. Arguments go empty through several paths (heredoc
expansion, a mistyped call), and `cd` with an empty argument does nothing
and does not fail.

**How to apply:**
- Start scripts with `set -u` and **validate the target directory against
  a pattern** before acting (`case "$P" in */scratchpad/e2e-*) ;; *) exit 1;;
  esac`). `cd "$P" || exit 1` alone does not stop an empty argument.
- Generate scripts through a quoted heredoc (`<<'EOF'`) or a Python raw
  string, and `cat` the result once to see the `$` still there.
- **Pin the required sections of agent-facing briefing files (`AGENTS.md`
  and kin) in `make check`.** A check that fails the moment a section
  disappears beats noticing in review.
- Look at `git diff --stat` before committing. Hundreds of deleted lines in
  a documentation file is a signal an intended change almost never sends.


## A cross-platform test harness detects its own defects first

**Symptom:** A harness was built to run Linux tests in a container from a
macOS/arm64 development host, and pointed at the 21 repositories carrying GOOS
branches. Five came back "passes on macOS, fails on Linux" — and three of those
were caused by the harness's own defaults.

- A locally cached amd64 image was picked silently, so **every repository ran
  under qemu**. The only notice was one line, `image platform (linux/amd64) does
  not match`, and the symptom was `qemu: uncaught target signal 11 (Segmentation
  fault)` — which reads exactly like a real crash.
- The container ran as root, which **bypasses DAC**. A test that chmods a path to
  0500 and expects the write to fail saw it succeed, and failed.
- The official golang image pins `GOTOOLCHAIN=local`, so a repository whose
  go.mod asks for a newer Go appeared as "fails on Linux".

**Why:** All three are defaults that silently do something else, and **a defect in
the harness wears exactly the same face as a defect in the code under test**. The
moment a difference appears it reads as "found a platform-specific bug", and the
hunt for a bug that does not exist begins.

**How to apply:**
- Pass `--platform linux/<arch>` explicitly. A wrong-architecture image announces
  itself with a single warning line.
- Under rootless podman, run as the invoking user with `--userns=keep-id`.
  **Permission-dependent tests are meaningless as root.**
- Pass `GOTOOLCHAIN=auto`; do not let the image's Go version judge go.mod.
- **Run both platforms and diff them.** One side alone makes a harness defect look
  like a defect in the target. Harness defects surface as many repositories
  failing at once, so the breadth of the difference is the first thing to read.
- When a difference appears, repeat it and measure the flake rate. Do not call it
  platform-specific from one run: in this case the two survivors were 3-of-5 and
  3-of-3, a race and a deterministic defect respectively.
- Keep build and module caches in named volumes, not on the bind mount — virtiofs
  is slow there and unix sockets do not traverse it.
- qemu writes core dumps into the bind-mounted source tree (100MB+ each). Check
  the working tree is still clean after a run.


### An independent pass finds its real defects right after a fix — the place of the fix is the place of the next defect

**Symptom:** One release of a CLI agent (2026-09; 68 files, three ADRs) went
through five full independent verification passes and one narrowed one before
the tag. **Every pass surfaced a real defect that the previous fix had
introduced**: fixing "exclusions are not saved" produced "turning a server back
on discards hand-written exclusions" (caused by the list the fix introduced);
fixing that produced "the settings panel shows stale values"; the one line added
there was wrong in two of the four states it described. In parallel, fourteen
assertions in the ADRs were false — a command that does not exist, a persistence
model that was never implemented, work listed under Consequences that was not
done, a generated document said to show the banner that did not.

**Why:** A fix does not repair verified ground; it **creates a new surface**,
and nobody has looked at that surface yet. The author cannot doubt their own
assertions — having read the document is no defence. Touching the tree while a
review runs forces the reviewer to re-verify a moving target, and the findings
come back as a mix of "already fixed" and "still there", which degrades the next
pass. Meanwhile severity fell monotonically (feature broken → data loss → stale
display → wording in one state). That trend is the evidence of convergence, and
it decides between "one more full pass" and "narrow it and finish".

**How to apply:**
- Ask the review brief explicitly: **what did the previous fix bring in?** Have
  the fix commit read as the change it is, not as a change assumed correct.
- **Mechanically check every assertion** in ADRs, READMEs and the CHANGELOG
  before committing: grep every named `/command`, subcommand, `[section].key`,
  type name and `$ENV` against the source (138 claims in that release). This
  only proves the names exist — charge the reviewer with "X shows Y" as a
  question of meaning, not of names.
- When you replace a name to make the check pass, ask whether **the replacement
  carries the same fact**. Swapping a command that does not exist for one that
  exists but does not carry the fact passes the check and makes the sentence
  false.
- **Do not touch the tree while a review runs.** Wait for the findings, fix them
  together, and send the next pass.
- Judge convergence by **the trend of severity**. When the worst finding has
  fallen to "wording in one state", run one short pass narrowed to the last
  commit; if it comes back empty, tag.
- Keep operator wording **mode-neutral**. A word that is false in one of the
  interactive TUI, the non-interactive REPL and one-shot mode ("asks",
  "footer", "`/auto`") is replaced by one true in all of them ("gated"). Every
  recurring wording defect had this shape.

## A gate that only ever runs by hand needs a test of its own

**Symptom:** A release-verification target shipped in a state where it absorbed both an unpack
failure and a binary that would not run, and still printed OK. The comment directly above that
target recorded an earlier incident — a notarization probe failing open, letting an un-notarized
zip ship green. The same shape was repeated on the next line.

**Why:** The gate runs once, by hand, at release time, and is normally given **valid** input.
There is structurally no occasion on which it meets a bad artifact, so it can break and stay
green. Writing the incident down does not exercise the gate.

**How to apply:** Write a self-test for the gate and put it in the routine check (`make check` or
equivalent). Feed it synthetic bad input — a corrupt artifact, a binary that will not run, a
binary from another version, a missing marker — and assert a **non-zero exit**. Include one valid
case too, so a gate that rejects everything is not mistaken for a working one. Run the self-test
against the unfixed gate first and confirm it reports the misses, before trusting it.

### A model-comparison bench interleaves configurations per case, time-boxes every call, and prints progress live

**Symptom:** Three thinking configurations of the same model, run in
parallel, throttled each other: medians above 30 s against a measured 4 s
in production, useless for comparison. Run sequentially instead, one call
stalled for 515 s, ate the 25-minute budget, and three of four
configurations ended with zero data. Output sat in `t.Log`'s buffer, so
nothing was visible for 25 minutes (2026-09, choosing an agent runtime's
risk-evaluation model).

**Why:** API conditions — congestion, retries, stalls — change by the
minute. Running each configuration as a block mixes time-of-day into the
between-configuration difference. A single overall deadline lets one stall
take everything after it down. Buffered output cannot distinguish "not
finished" from "broken".

**How to apply:**
- **Cases outside, configurations inside**: interleave, so a swing in API
  conditions hits every configuration alike.
- **`context.WithTimeout` per call** (say 90 s); a stall is one ERR line
  and the loop moves on. The overall deadline sits above that.
- **`fmt.Printf` each decision as it lands**, timestamped; keep `t.Log` for
  the end summary. `go test -v` shows logs only when the test ends.
- Do not run the same model in parallel with itself; parallelise across
  models only.
- Report both per-decision (several rounds folded into one verdict) and
  per-call numbers, and count retries.

### Measure an inline TUI's row arithmetic in tmux and script — the renderer paints only the latest View per tick

**Symptom:** With Bubble Tea's inline renderer, closing a panel left one
line of earlier output standing at the top of the screen. Two fixes
reasoned from the code (pad the frame to height-1; choose the layout per
View from the rows remaining) passed the unit tests and did nothing on a
real terminal. A pseudo-terminal of known size (`tmux new-session -d -x
100 -y 30`, driven with `send-keys`, read with `capture-pane -p`) plus a
raw byte recording under `script` settled it: (1) a frame of N rows
drawn from row printed+1 scrolls the terminal by printed+N−height rows,
so a frame of height-1 leaves one row; (2) the renderer buffers the View
of every Update and paints only the latest one on its tick — the
full-height frame healed the counter, the next View re-derived the
layout from the healed counter and flipped to the shorter frame, and
the shorter frame is what reached the terminal (2026-09, an agent's
TUI).

**Why:** The model's row accounting assumes the View it returned is the
one drawn, but a later Update can replace it before the tick. State
re-derived on every View diverges when an update is self-referential,
like a self-heal. Unit tests see only View's return value and cannot
detect the difference.

**How to apply:**
- Fixes that touch row positions or scroll amounts are **counted on a
  real terminal**: tmux (fixed size, `capture-pane`) and `script` (raw
  bytes: count the `<CSI nA>` cursor-ups and the `\r\n`s). Eyeballing a
  screenshot comes last.
- A layout choice (full screen vs. fit in the rows remaining) is **made
  once when the panel opens and held until it closes**, never re-derived
  from the counter inside View.
- A trace of "lines the model returned" is not evidence of what was
  painted; confirm painting from the raw bytes.

### Anything that occupies N rows in an inline TUI must erase below itself — the renderer's flush clears one

**Symptom:** Inline terminal images in an inline TUI, with the row
accounting already correct (the emitter declares a box and tells the row
counter that number; the unit tests pass). On a real terminal three images
still left three footer fragments standing on the screen. Neither the unit
tests nor four independent verification passes reading the source found it
(2026-09, an agent's TUI).

**Why:** Bubble Tea's inline renderer flushes a queued line from the top of
its own frame and appends `EraseLineRight`, which clears **one row, to the
right of the cursor**. The image then draws down N rows at its declared
width, so every cell of the old frame to the RIGHT of a narrower picture
survives on every row the picture covers. A 40-column picture under a wider
footer leaves the difference on every one of those rows. This is
independent of the row arithmetic and happens with the accounting perfectly
correct.

**How to apply:**
- Under an inline renderer, output that occupies **more than one row**
  (images, sixel, hand-drawn boxes) writes one erase-to-end-of-screen
  (`ESC[J`) immediately before its payload. Nothing above the cursor is
  touched, and the renderer repaints just below it anyway.
- "If the row count is right the screen is right" does not hold: the
  declared **width** is occupied too, and erasing happens per row.
- This class is **decidable only on a real terminal**. No number of
  source-reading passes reaches it. Take an A/B on the real terminal — the
  fix, and the same tree with the one line removed — under the same
  conditions, and count the leftovers.

### A terminal capability probe asks a second question every terminal answers

**Symptom:** A probe asked whether the terminal could draw inline images
with an APC graphics query and read silence as "no" via a timeout. On Apple
Terminal it blocked for the **full 2.001 s at every start**, and because
that terminal does not understand APC it printed the query's own body —
`Gi=31,s=1,v=1,a=q,t=d,f=24;AAAA` — on the operator's screen first (2026-09,
an agent's TUI; the environment classified only tmux/screen, the kitty
family and iTerm2, so every other terminal took this path).

**Why:** The capability question has **no negative answer**. A terminal that
does not know the protocol simply says nothing, and silence cannot be told
from slowness, so the only usable verdict is the timeout — which every
terminal that cannot draw then pays in full.

**How to apply:**
- Send a **DA1 request (`ESC[c`) immediately after** the capability query.
  Every VT-compatible terminal answers it, so "a DA1 reply arrived with no
  graphics reply before it" is a definitive no. Measured: 2.001 s → under
  1 ms. Keep the timeout as the backstop for a terminal that answers
  neither.
- A terminal that does not parse the escape **prints its body**. Erase the
  line the probe dirtied before handing the terminal to the UI (`\r` plus a
  line erase).
- Record the effect by **measuring before and after with the same
  instrument**. Both the probe's duration and the dirtied screen are
  observable only on a real terminal.

## Do not write a single observation as a rule — write it with its counts

**Symptom:** A search API's answers came back without citations twice when
the reply language was `ja`, and README, the manual and a note attached to
results all asserted "citations are emitted only for `language=en`". The
third `ja` run returned 24 citations. Five runs read en → 29, 29 / ja → 0, 0,
24.

**Why:** Two agreeing observations look like a rule, and isolating the cause
(country vs language) one variable at a time only added confidence. But the
system was non-deterministic (an LLM sits behind the API) and two samples
were far too few. The assertion spread at once to every reader-facing surface
— manual, README, result note — and the correction had to reach all of them.

**How to apply:**
- For non-deterministic systems (LLMs, search, the content of external
  responses), **record the count and the breakdown verbatim** ("5 runs: en
  29, 29 / ja 0, 0, 24"). "Only when" and "always" wait for ten or more
  agreeing runs on the same input.
- Reader-facing wording follows the observation ("unreliable for non-English
  replies"). A note attached to a result does the same, and adds the sentence
  that keeps an empty value from being read as "does not exist".
- Isolating experiments (one variable at a time) narrow the candidate causes;
  they **do not prove a rule**. Repetition under identical conditions comes
  after the isolation, not instead of it.
- When correcting an assertion, grep every surface that carried it (README,
  manual, result note, ADR, CHANGELOG) and fix them in one pass; fixing one
  surface creates drift.

### When you add a boundary, write the test that drives the real product through the real boundary — stubs and permission stand-ins pass while production is broken

**Symptom:** The file tools' reads were moved into a child process under a purpose-built profile that denies the credential list. Unit tests and the full gate were green and it shipped; independent review measured a Critical. **One `.env` in a project made `search_files` answer "no matches (0 files scanned)"** — a false negative with no error and no warning. Four High findings followed from the same pass: reads silently truncated at 85 KB, `file_info` collapsing a permission error into "not found", a cancelled walk losing its partial result, and the note the design promised being unreachable.

**Why:** No test **drove the real thing**. Every test stubbed the child (injecting a function) and simulated the kernel's refusal with `chmod 000`. `chmod 000` denies `open` and leaves `lstat` working; Seatbelt's `file-read*` denies both. That single difference was the defect, and the stand-in pinned a path production never takes. The kernel-level test ran `/bin/bash`, never the Go child that actually runs, and the startup probe did the same — so it passed while every walk in the project was broken.

**How to apply:**
- When you add a boundary (a sandbox, a separate process, a different privilege), write in the same commit a test that drives **the shipped artifact through the shipped boundary**: build the binary, run it under the real profile, feed it a real request. Slow is fine (2–4 s here).
- **Distrust permission stand-ins.** `chmod`, not-running-as-root and a severed network each deny a *different set of operations* than a kernel deny does. If you cannot state the difference between the stand-in and the real thing in one line, you cannot use the stand-in.
- **Verify the startup probe with the operation the runtime actually performs.** A probe that uses a different process, profile or library stays green through a production break. Make it walk the same sequence, listings included.
- **Drive the path that TURNS THE BOUNDARY ON, in its production form.** The next release of this same design did write a test driving the real artifact through the real cage — and never ran the function that *installs* the cage. The probe was asking the child for a tool outside its set, so it failed on every start, the cage was never installed, and the whole feature shipped inert. A test must pin not only "does it behave correctly inside the boundary" but "does the boundary go up at all". The second is a single path — installer, wiring, startup verification — that exists only in production.
- Say in the stubbed tests that they are **not the production path**, and name the test that covers it.

Related: "independent review turns up something real right after a fix". For the same reason a fix creates a new unverified surface, **a new boundary is at its least verified the moment it is added**.

### If a design document says "applied at every N", write the test that enumerates N in the same commit

**Symptom:** An agent runtime's ADR said one function is applied at **every spawn site**, so a child process cannot be added without inheriting the rule. The rule: the runtime's own configuration variables (an API key among them) reach no child. The build was green, and an AST test pinning the namespace partition passed. Four kinds of child did not apply the rule: the shell of a sandbox-disabled launch (where the API key did reach it), two kinds of startup verification probe, and the clipboard capture. It surfaced not months later but the same day, during unrelated work.

**Why:** The existing test pinned a **list of names** — which variables are "mine" and which are "exported for children" — not the **coverage of call sites**. However rigorous the name partition is, it never looks at an `exec.Command` that forgot to call the function. "Every X" in a design document means the set of X its author knew at the time, not every X in the code. The two diverge from the moment the sentence is written.

**How to apply:**
- When a design document says "applied at every X" or "X always goes through Y", put a test that **enumerates X and checks Y** in the same commit. If you cannot write one, weaken the sentence to "as of today".
- Write it against the AST so it closes the **class**: fail when a function that builds an `exec.Cmd` does not name the helper, fail when a path-handling package calls `os.Open` directly. If you use an allowlist of reviewed sites instead, make an **unused allowlist entry fail too**, so a site that disappeared does not linger as debris.
- Put exceptions in the code with their reason. In the example above the legitimate exception was the bench harness, which **launches** the runtime — a parent, not a child it spawns. Choose exclusions by reason, not by directory name.
- When fixing, look for a line that **undoes the fix** right after it. Two lines below the probe's new rule, the code rebuilt the environment from the parent's to set a temporary directory; left alone, the fix would have died in the commit that made it.

Related: "drive the path that turns the boundary on". That one pins whether the boundary is installed at all; this one pins whether it is installed at **every** entrance.

### A withdrawn mechanism disappears from the code, not from the **prose** — write a test that compiles the strings the model reads

**Symptom:** File-mediated results were removed from two servers and released. The
implementation, the schemas and the tests were all correct, and an arch test pinning the
`work_dir` spelling passed. The shipped artifacts still told the model, in `run_query`'s
**description**, that large results are written as JSONL under `workspace_root`, and in
`get_usage`'s error table, to retry with `workspace_root` or raise
`inline_row_threshold` — both operations the server now rejects. On another server the
container manifest returned by `describe_runtime` still said results are files under
`/work`; on a third, the setup guide still told the reader to fill in a config key that
had been deleted. The same sweep turned up a server whose schema declared `work_dir`
**optional** while the handler required it: the model reads that it may omit the
argument, and the call then fails.

**Why:** The arch test pinned **identifiers** — the spelling inside a schema, the presence
of `required` — not the sentence next to them. Prose is not compiled: it has no type and
no reference, so removing a mechanism leaves the paragraph describing it untouched, and
nothing prompts anyone to grep. In MCP that paragraph is not a comment but **part of the
product**: the `tools/list` description, the `get_usage` body and the `describe_runtime`
manifest are inputs the model reads and acts on, so a lie there becomes a wrong call.
Build-tagged suites (`integration`, `e2e`) compound it: `go test ./...` never compiles
them, so a field deleted by the withdrawal leaves them broken for months.

**How to apply:**
- When you withdraw a mechanism, make a list of the retired terms and, in the same
  commit, write a test that walks every string the model reads — each tool's description
  and input schema, the usage document, the runtime manifest — and fails on any of them.
  Keep one list, not one for schemas and one for prose.
- Pin the **shape** of the contract too, not just the identifiers: "a tool that declares
  `work_dir` must also require it". An optional mandatory argument reads, to a model, as
  one with a default.
- Add `go vet -tags <each tag> ./...` to `make test`. A tagged suite you cannot run can
  still be type-checked; needing real hardware to execute is not a reason to let it rot
  uncompiled.
- Count the sweep in **surfaces the user and the model read**, not in code. The
  checklist: the `initialize` `instructions` field (the first thing the model reads —
  it arrives before `tools/list`), serverInfo, each tool's description and input schema,
  the usage document, the runtime manifest, error messages and their details, `--help`,
  READMEs (**every language**), setup guides, config examples, the RFP, and any
  **bundled skill or prompt you ship**. The last two are the easiest to forget: a skill
  is *installed into the agent's skill directory and read as instruction*, so a retired
  argument name left there is not a documentation wart — it is the agent being told to
  make a call the server refuses. Fixing only one language's README is the likeliest miss.
- Check whether the existing tests only assert that things are **present**. The bundled
  skill's tests checked that every tool name appeared in it, so a retired argument
  surviving in eight files failed nothing. A withdrawal needs **absence** tests.
- Do not rewrite historical design documents; annotate the withdrawal in place with a
  strike-through, a date and the successor. History stays honest and stops reading as
  current.

Related: "If a design document says a rule applies to all N, write the test that
enumerates N in the same commit." That one pins the coverage of the application points;
this one pins the coverage of the **explanations**.

### Verify WebKit authorization at the actual callback boundary

**Symptom:** During a macOS client port, JavaScript navigation produced click-like
metadata. An async method with a name resembling an authentication delegate
compiled successfully but never received the certificate challenge.

**Why:** Navigation type and button number do not prove human approval. A method
that merely resembles an optional Objective-C protocol method can compile without
implementing it. Isolated predicates cannot reveal missing call paths for redirect
cookies or certificate verification.

**How to apply:** Bind web-to-native connection authority to a one-shot native
confirmation of an immutable candidate. Use a temporary HTTPS server and real
WKWebView to exercise automatic location changes, form POSTs, cookies received
after redirects, certificate challenges, and replay after cancellation. Trust
fixture certificates only inside the test; do not alter the system trust store.
Check the SDK's async delegate name and do not equate successful compilation with
callback delivery. If a parallel dependency suite stalls shared fixtures, rerun
sequentially to diagnose it rather than deleting failed tests.

Source: spice-client port and simulation verification (2026-09).

## Cut an interval at a boundary on "touches it", not only on "straddles it" — a miss hides behind sampling phase

**What happened:** An aggregation filing activity into logical days (a day
starting at 05:00, say) looked for the day-boundary cut with
`seg.Start < boundary && boundary < seg.End` — a segment straddling it. Segments
are contiguous, so when a state change or a gap split lands exactly on the hour,
**nothing straddles the boundary**. No cut happened, and because the next
boundary was only recomputed once the stored one was cleared, **a boundary
already passed stayed as the limit and every later boundary was skipped too**.
A session crossing days then ran until the next long absence, and the day it
should have opened reported zero. This also disabled an already-released cap
("a session spans at most two logical days") by the same route.

**Why it stayed hidden:** Whether it fires is decided by the **phase** of the
sampling tick (`unix % interval == 0`). The tests used a single phase, and in a real multi-week history almost no
sample ever met the condition — so the deployment was asymptomatic. The phase is re-rolled every time the daemon
restarts, so "not happening now" is not "cannot happen". A code-reading
verification pass also failed to see that the predicate tested only the straddle.

**How to apply:**
- Make the predicate **two-pronged**: cut on a straddling segment *and* on a
  segment that begins **at or after** the boundary (`!start.Before(limit)`). In a
  contiguous series the second is the common case; the first alone always misses
  the exact touch.
- For any value held until a condition is met (a limit, a next deadline, a next
  boundary), ask whether it **recovers from a miss**. If it does not, put the
  code that advances it next to the test that consumes it — one miss otherwise
  becomes a permanent outage.
- Test this class with a **table over step sizes and start offsets (phases)**.
  A test written at one phase cannot detect it, structurally.
- If the boundary is a wall-clock hour, **daylight saving makes the logical day
  23 or 25 hours**. Before writing "never exceeds 24h", write the test that
  crosses the change and measure what is conserved and where the boundary lands.

## A counter fed by the view's draw callback proves nothing in a headless test

**What happened:** A live-peer gate for a SPICE client asserted that the session's
`frames_presented` diagnostic grew once a real server was sending frames. The
first run waited 30 s and failed: that counter is incremented in the Metal view's
drawable callback, and a `swift test` process has no view. The frames were
arriving; nothing was counting them. The design review had predicted it from the
source before the run reproduced it (spice-client ADR-0002, 2026-09-18).

**How to apply:** In a headless test, observe what the view observes, not what
the view reports: subscribe to the frame source with the demand a visible window
would set and count distinct revisions. Do the same audit for every counter a
gate asserts — find the one call site that increments it and ask whether that
code runs in the test process at all. And when a suite is enabled by an
environment variable, `swift test` exits 0 with the suite skipped if the variable
is missing; a gate that trusts the exit status alone passes with zero tests, so
have the tests leave a receipt the gate checks.

## A server that serves one client at a time turns parallel test suites into flakes

**What happened:** A live-peer gate grew a second test suite. Both suites drove
the same QEMU SPICE server, and `swift test` runs suites in parallel. Connections
started failing, a frame observer starved, and the failures moved between tests
from run to run. Nothing was wrong with either suite: QEMU's SPICE server serves
one client, and the two suites were fighting over it (spice-client, 2026-09-18).

**How to apply:** Before adding a second suite against a shared live fixture, ask
how many clients the fixture serves at once. When the answer is one, `.serialized`
on each suite is not enough — it serialises within a suite, not across suites. Run
them as separate sequential invocations of the test runner, which is the only
arrangement the runner cannot undo. The symptom to recognise is failures that
migrate between tests across runs while each test passes when run alone.

## Pin a defect you are not fixing as a checked fact, so the gate fails when it is fixed

**What happened:** A live gate found a real product defect: after the first
viewport resize, no later resize on the same agent connection reached the guest.
Fixing it meant widening a vendored dependency's patch, which an ADR had scoped
deliberately, so it was not this change's call. Dropping the assertion would have
left nothing to notice the day it was fixed (spice-client, 2026-09-18).

**How to apply:** Record the defect as the assertion. The gate requires the
guest to apply the first mode and requires the second mode to be *absent*, with a
message saying that its arrival means the defect is fixed and the test and the
records must be updated. A defect pinned this way cannot rot quietly in either
direction: it fails if it gets worse, and it fails if it gets better while the
documents still claim it is broken. This is the machine-checked half of "record
what you did not fix".

## A stall at a round number on a credit-controlled channel is a window problem, not a data-handling bug

**Symptom:** Sending a file to a guest agent stopped at exactly 32,000 of 64,000 bytes,
every run. The partial content was correct, nothing errored, nothing disconnected. The
chunk size was the dependency's 16,000-byte default.

**Why:** The channel was under token (credit) flow control. A message costs one token per
2 KiB wire fragment; the peer grants ten tokens and returns them five at a time. A
16,000-byte chunk needs eight. The first one spends eight of ten leaving two, the batch
of five arrives making seven, and the second chunk needs eight — for ever. The stop
position is the same round number every run because it is decided by the window's
arithmetic, not by the data. Reading the byte handling and the buffer boundaries finds
nothing, because there is nothing there to find.

**How to apply:**

- A stall at a position that is constant across runs and an exact multiple of the chunk
  size points at flow control. A data-dependent bug moves with the content.
- Measure the credits first: initial token count, the unit a message is charged in, and
  the granularity of replenishment. Three numbers, and the implementation's log gives
  them faster and more reliably than the specification.
- Choose the chunk so it **fits the window the peer can open**, not so it is as large as
  possible. If one message costs more than half the initial credit, the second one jams.
- A dependency's default is a default for the peer that dependency had in mind. Pointing
  it at a different hypervisor or server implementation voids the assumption with it.
  Pass the number explicitly as your application's, with the reason recorded.

## Size stability is not a completion signal when the writer preallocates

**Symptom:** A guest-side watcher observed the receipt of a transferred file by hashing
it "once its size has been stable for two seconds". It returned the hash of a partial
file.

**Why:** The receiving daemon allocated the whole file before the first byte arrived, so
a file still being written already has its final size. "The size is stable" is true from
the moment the transfer starts, which makes it a check that asserts nothing.

**How to apply:** Key on a quantity that tracks the writing. Modification time is not
affected by preallocation. Always pair it with:

- **A hard deadline that always emits a line.** Silence does not distinguish "still
  writing" from "never coming", so a quiet observer leaves the verifier waiting.
- **A line for the file being removed.** Cleaning up after a completed transfer is
  common, and a vanished file should be a reported outcome rather than silence.
- **A digest computed by the receiver** as the evidence of completion. The sender's
  "succeeded" is a statement about the sender's own state.

## A gate that cannot name the layer it observes has only proved the layer below

**Symptom:** A live-peer gate verified on every run that an injected key reached the
guest, and passed throughout. The day someone gave that guest a desktop and typed into
a terminal on it, **not one character arrived.**

**Why:** The test read the guest's `/dev/input/event*` directly. The key really did
reach the guest kernel. But the guest's X server had no input driver and no udev, so it
started with zero input devices and no X client ever saw a keystroke. What the test
proved was "reaches the kernel"; what a user experiences is "reaches the application".
A whole layer sat between the two, and it had been broken for as long as the gate had
existed.

The observation point ends up one layer low because that is the easy place to put it.
Raw input events are easy to read; observing an X client's receipt needs another
program. **The convenient observation point is usually below the real one.**

**How to apply:**

- Write down, per test, which layer it proves. If you cannot, its scope is unknown.
- When the layer you observe is below the layer a user experiences, **add a check that
  covers the gap**. Not everything has to move up. Here the guest was made to log how
  many input devices Xorg took, and the gate requires at least two: key delivery is
  still observed low, but the missing layer now fails the gate.
- A check that has never failed is evidence of correctness *and* evidence that it may
  be looking at nothing. Break it deliberately once and confirm it fails.
- Paths a person experiences — drawing, input — get **looked at by a person at least
  once**. Both defects found here surfaced on the first day someone looked at the
  screen, with every automated check green.
- **The observer has to be running before the event it observes.** The trap waiting
  right after you move an observation point up: the X key observer was started after
  the readiness marker the harness waits on, so the injected key arrived first and was
  missed. Start it before that marker and confirm it is up before emitting it.

## The script that reads the trace calls a working fix broken

**Symptom:** a GUI defect was fixed and checked against an event trace from the real machine
(mouse-downs, actions, menu tracking and settings changes, to the millisecond). Twice in one
day the matching script declared a correct fix a failure: first "29 of 58 clicks did nothing",
then "5 of 7 selections were lost". Both times the person who had used the app said it looked
fine. (net-meter, 2026-09)

**Why:** both were assumptions about **the order of the record**.

- First: the recorder's monitor is called **after** the app's own. By the time its line for a
  closing click is written, the panel is already closed. Read as "the state before the click",
  that made every closing click look as if nothing had happened.
- Second: the binding's setter runs about 190 ms **before** the menu's end notification. The
  script looked only at the lines after it.

Reading the raw lines in time order settled each in minutes.

**How to apply:**
- Group one physical action under one identifier (an event number, say) and judge it by **the
  state it left behind** (the state at mouse-up, say). Do not take the order of lines in the
  record for the order of cause and effect.
- Before writing a match of the form "B must follow A", check the actual order in the raw
  record once.
- **When the person who used it and the script disagree, read the raw lines first.** The
  script is code you have just written and nobody has reviewed.
- Keep the recorder behind a compile-time flag and confirm **by symbols** that it is not in
  the release binary (`nm <binary> | grep -ci trace` gave 0 for the release and 25 for the
  diagnostic build). This entry first recommended `strings`, which was wrong: Swift stores ASCII
  string literals of 15 bytes or fewer inline in the code, so they show up in neither `strings`
  nor `grep -a`. The diagnostic binary also gave 0, so the check never established absence
  (corrected 2026-09-20). **A check for absence must first be seen to come up positive on a
  target where the thing is present.**

## Check a guard by mutating a copy — "not caught" comes in three kinds

**Symptom:** Guards that tests had been green over from the first run were broken one at a time
to see whether the tests noticed (90 mutations over four rounds). Most were caught. Some rounds
had void mutations that merely "failed" — on a compile error or an endless wait — and in the
last round 3 of 35 were not caught, each pointing at a different defect: (1) a backstop that
shortens long strings never reached a string held directly by a `map[string]any` — **a hole in
the guard itself**. (2) a test paging a very large document always passed an `offset`, so it
never made the call a reader makes first (no `offset` — the path that used to return the whole
document) — **the test did not take the real path**. (3) with the one-pass sanitising removed,
per-site escaping was still there and the output stayed inert — **a redundant mechanism hid the
absence**.

**Why:** A green test shows that the guard is there. If it stays green with the guard removed,
it was looking at nothing. Reading the code does not settle it: the author follows the path
the author had in mind.

**How to apply:**
- Mutate **a copy, never the working tree**. Copy the tree into a fresh temporary directory
  per mutation and delete nothing (cleaning up is a person's decision, later). The real tree
  is never touched.
- Confirm by file hash that the mutation **was applied**. A pattern that did not match is
  "void", not "caught".
- Confirm the failure on **an assertion line** (pick up `_test.go:NN:`). A compile error, a
  panic or a timeout is void — rewrite a mutation that leaves an import unused so that it still
  uses it, and move the arithmetic check ahead of the path that would panic.
- A guard must fail, not hang. A test that waits forever because a fake clock never advances
  gets a runaway cap so that it fails fast. Always run with a timeout.
- When a mutation is not caught, decide which of the three kinds it is before fixing: a hole →
  fix the guard; a path mismatch → make the test call what a reader calls; redundancy → reduce
  to one mechanism (do not leave it as an "equivalent mutant" — the reader can no longer tell
  which one is load-bearing).
- Rerun all of them after fixing, and record "caught N / blind 0 / void 0".

## A release gate that skips is a gate that passes

**Symptom:** A live e2e test measuring drift between an upstream API and the primary sources (a
government catalogue, a standards body's scores) served as a release gate — and called
`t.Skip` when a primary source was unreachable, and only `t.Log` when one half could not be
compared. Until the independent review said so, nobody had noticed that "compared nothing" was
green. And since plain `go test` prints no log for a passing test, a run that compared 8 of 8
looked the same as a run that compared none.

**Why:** A skip says "this test does not apply in this environment". For a gate, "could not
check" does not mean "does not apply"; it means **not checked**. It is fail-open in a shape
other than `|| true`.

**How to apply:**
- In a test that runs as a gate, an unreachable dependency, a missing artifact and a fixture
  that lost its meaning (dropped from the tracked set, say) are all **failures**, with a
  message that says the check did not run.
- Log the evidence of the comparison (how many of how many agreed) and run the gate's target
  so that it shows (`go test -v`). A gate whose evidence is invisible cannot be told from one
  that compared nothing.
- Demonstrate the gate failing once: falsify an expectation and watch it go red.

## Write a drift check against primary sources with tolerances — exact equality fails on sync lag every time

**Symptom:** The design watched, in e2e, whether an aggregating API's data (catalogue listings,
daily model scores) kept agreeing with the primary sources. Written as exact equality it fails
without fail: on entries added to the primary source that day, and at the boundary of the
model's daily refresh.

**How to apply:**
- Derive the tolerances **from how the sync works**: sample only entries added more than N
  hours ago; require "entries older than N hours ≤ upstream's total ≤ the primary source's
  total"; accept a daily-model value that equals any of the last three model dates. Fail only
  above a set share of the sample (a quarter, say).
- Draw the sample from both ends, the newest and the oldest. Lag shows at the new end, rot at
  the old one.
- Put into the failure message the design decision it reopens ("reconsider the declined
  primary-source cross-check layer").
- Share **one engine, and so one pacer**, across the whole e2e suite. A client per test
  multiplies the burst allowance by the number of tests and runs into a per-IP ceiling.

## The write path turns an escape into the raw character — close it by checking every source file

**Symptom:** Every time a fixture for "upstream sends an escape sequence" was written, the path
between the author and the file (an agent's file-writing tool, the decoding of a shell
argument) turned the typed backslash + u + four digits into **the raw character**. It happened
three times. The first put a raw ESC into a JSON fixture inside a Go raw string: invalid JSON,
and a test that passed without having tested anything. The third put a raw U+202E into the
source of a test about bidirectional controls — the very shape of a "Trojan Source" change.
The document warning about all this carried a raw ESC itself.

**Why:** Somewhere along the path, one JSON-style unescape too many. It cannot be seen (editors
do not show control characters), so care does not prevent it. The same kind three times over
gets a structural fix.

**How to apply:**
- Put a test at the repository root that walks every source file (code, Markdown, config,
  scripts) and refuses a raw C0 control (other than tab and newline), DEL, C1, a bidirectional
  control, U+FFFD and a BOM. Assert a floor on the number of files walked, so that it cannot
  go green having looked at nothing.
- When it fails, do not retype. Replace with a script that builds the code points with
  `chr()` — and holds neither the characters nor the escapes itself.
- Build fixtures that carry escapes from double-quoted strings, and assert first **that the
  text arrived**. A test firing blanks over a broken fixture is green with or without the
  guard.

## A fake that answers every question the same way cannot see a wrong question

**Symptom:** three defects in one audit had passed their suites for the same reason. A fake
container runtime answered every lookup with one canned id, so a delete that looked up a name
the create path never uses stayed green for several releases. A slash command's test handed it
a fake resolver, proving only that the argument reached the layer that then lost it. A
preferences stub failed by throwing *before* changing anything, while the OS fails *after* — so
every test of "what happens after a failed write" ran against a failure that does not occur,
and a relaunch was modelled by reusing the same object, which kept the very state a relaunch
discards.

**How to apply:**
- A fake keeps state and matches the way the real thing does; a canned reply is a test of the
  caller's control flow and nothing else.
- Name the layer a test observes. If the defect is one layer down, the test is a test of the
  layer above — add one that goes through.
- A stub's failure modes come from the real component's, not from what is easy to write.
  Where both "nothing changed" and "changed, then failed" exist, run every case both ways.
- A process boundary in a test is a **new** object over the same persistent stubs.
- A refusal test needs a positive control: the same flags with a valid value must be accepted,
  or a mistyped flag name passes as a refusal (the flag package refuses it with the same exit
  status).
- When a repair is for a class, mutate a scratch copy per clause and watch the named test
  fail. Four of four did here, and two earlier "fixes" that reviews later withdrew had tests
  that passed against the behaviour they were meant to exclude.
