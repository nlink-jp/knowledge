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
