# Development-Process Judgment

Lessons about **how to decide**, not how to build — rewrite vs refactor,
external contributions, ADR granularity, documentation practice. Each entry
follows **symptom → why → how to apply**. Prescriptive process (ADR-first, small
typed commits) lives in CONVENTIONS.md.

---

## Design & implementation decisions

### Rewrite or refactor — decide by three conditions

**Symptom:** A tool that was one 636-line main.go was faster to rewrite from its
spec than to refactor. Another tool, already split into files with moderate
coverage, only needed refactoring.

**Criteria for rewriting (consider when all three hold):**
- One giant file (600+ lines) with global state referenced directly from
  functions
- Extremely low test coverage (<10%), black-box tests only
- Untestable error paths (`log.Fatal` scattered outside main, hardcoded I/O)

**Rewrite procedure:** extract the spec (README + code + test expectations) →
architecture doc → move current code to `legacy/` → **keep testdata and expected
outputs** (regression guarantee) → reimplement → all regressions pass → delete
legacy.

### Before building a workaround, check whether upstream already fixed it

**Symptom:** Against a pinned vendored dependency's limitation, a custom
converter was designed and implemented — but upstream had landed **the exact
same fix five days after the pin**. Bumping the submodule three commits made the
custom work entirely unnecessary (and the custom version had a gap that crashed
rendering).

**How to apply:** When blocked by a dependency: (1) check upstream's latest /
issues / PRs first (grep `git log HEAD..origin/master`); (2) even a fresh pin can
trail a fix by days; (3) assess bump side effects via header diffs; (4) bump the
submodule pointing at upstream (no fork needed).

### Structural drift calls for restructuring from first principles, not minimal patches

**Symptom:** For information hand-maintained in four parallel places, the
minimal "consolidate the lists" patch was proposed — and pushed back with
"reconsider assuming a full restructure". The eventual restructure removed the
very source of the parallel lists, and the drift stopped.

**How to apply:** When "drift keeps getting observed" or "the same symptom
recurs across versions", present the target end-state in 2–3 sentences instead
of a minimal fix and ask for a decision. Restructuring costs are recovered in
maintenance savings — but full restructures need approval; don't do them
unilaterally.

### Don't change defaults for a minority complaint — rescue via opt-in flag

**Symptom:** A "the default implicit behavior hurt me" report prompted strict
fixes, but (a) existing users weren't hurting, (b) a complete fix was
structurally impossible, (c) the reporter was an expert who notices the issue
immediately.

**How to apply (triage):**
1. Check whether anyone besides the reporter suffers under the current default.
   Zero → opt-in is the leading option.
2. Is the behavior destructive (data loss/crash) or recoverable (empty result,
   instantly noticeable)? Recoverable → lean opt-in.
3. Is a complete fix feasible? If holes remain, make it opt-in and don't promise
   "smart" behavior.

Design shape: CLI flag + config field (CLI > config > default), **default stays
compatible**, and expose an enum of three states from the start (smart /
compatible / no automatic handling) — a single opt-out flag proliferates into
per-input-path flags. Leave a "why the default is X" paragraph in the README.

### Breaking changes: confirm users exist → present options → get approval

**Symptom:** A config restructure proceeded without raising compatibility, and
the user had to interrupt to ask. The reverse failure too: deprecation code was
implemented and then deleted immediately because "there are no users" — written
and thrown away.

**How to apply:**
1. **Confirm real users first.** Even for released software, zero users means
   zero compatibility code.
2. With users, stop before changing user-facing interfaces and present options —
   clean break / migration shim / examples-only change.
3. Write code only after approval. Never write the compatibility code first.

### Migrations and integrations go regression-tests-first

**Symptom:** In a library-integration migration, writing regression tests for
current behavior first allowed accurately classifying post-migration differences
as improvements vs regressions.

**How to apply:** Write tests for current behavior → migrate → triage failures
into improvement/regression → update expectations for improvements, fix
regressions.

## Bulk & mechanical changes

### Verify "identical generated output" mechanically before sweeping vendored templates

**Symptom:** A template change was about to be mass-copied to 69 repositories
"because it's mechanical" — and was challenged: a conversion job that never
checks each project works is not acceptable. Writing the verification script
proved byte-identical old-vs-new output for 67 repos — the first actual grounds
for safety.

**How to apply:**
1. Run each repository's real generation command under old and new templates and
   diff (contrive dummy inputs to avoid full builds).
2. Sweep only after all-match (dry-run → apply). Exclude repos with detached
   HEADs or local changes.
3. Re-verify + org health check after the sweep.
- Pitfalls: archived repositories can't be pushed and drift forever — use the
  hosting side's archived state as truth and exclude them. Duplicate checkouts of
  one repo cause double commits.

### Bulk renames process longer identifiers first

**Symptom:** Running `register-object` → `register_object` first mutated
`sandbox-register-object` into `sandbox-register_object`, so the later
replacement found nothing — a silent rename gap.

**How to apply:** Sort targets by length descending (if one identifier is a
substring of another, the longer goes first). Finish with a grep for the old
pattern to confirm zero remnants. Separator swaps (kebab → snake) collide most
easily.

## External contributions

### Mine legitimate signal even from PRs you reject

**Symptom:** An AI-agent-generated PR bundled several changes plus the agent's
internal files — close-worthy — but contained **three legitimate hardening fixes
and a real bug report**. These were cleanly reimplemented and shipped.

**How to apply:**
1. Close with concrete reasons (scope mixing / AI artifacts); ask for splits of
   bundles.
2. Evaluate security/stability fixes and bug reports independently of the
   rejection; **rewrite adopted ones from scratch** (never copy PR code), with
   your own tests and rationale.
3. Credit adopted reports in commit messages and issues; close the loop
   afterwards ("your report shipped in vX.Y.Z").
4. Reporters are usually sincere — don't treat them as adversaries.

### Review behavior-describing doc PRs against the implementation

**Symptom:** A field-notes PR contained the **accurate, measured** statement
"if one object is quarantined the whole call fails; no partial success". Merging
it would have frozen a defect as documented spec. The implementation showed other
overage cases skip-and-continue while only unreadable objects aborted — the
asymmetry proved unintended design. The code was fixed first, the sentence
rewritten, then merged.

**How to apply:**
- Two-stage review question: ① does the description match actual behavior?
  ② **is that behavior intended?** Skipping ② promotes bugs into spec.
- Suspect asymmetries ("A skips, B is fatal") — usually omissions, not
  decisions.
- Fix order: **code → description → merge**. Merging first leaves wrong
  descriptions public.
- Behavior descriptions scatter across docs — grep the whole surface before
  fixing.

## ADR & release granularity

### Bundle small independent UX polish into one wide-scope ADR

**Symptom:** Five small improvements from live feedback (each <100 lines,
strictly additive) would, as individual ADRs, repeat the same Context five times
with thin Alternatives — ADRs that don't function as ADRs.

**How to apply:**
- **Bundle-able**: all strictly additive / no existing-path behavior changes /
  decision rationale shares one common ground.
- **Individual ADR**: independent design trade-offs / unique alternatives /
  likely to be searched for alone in six months. Any one criterion → separate.
- Bundle ADRs number their Decisions and keep per-item Alternatives (don't skip
  them). README/CHANGELOG unbundle into per-item entries. One bundle ADR ≈ one
  minor release.
- Counter-pattern: a breaking change in the mix forbids bundling (split into a
  polish ADR + a breaking ADR).

### Co-reinforcing design changes ship together

**Symptom:** Two ADRs treating different amplifiers of the same symptom were
discovered, with one's features depending on the other's output signals.
Shipped separately, neither changes the felt behavior alone.

**How to apply:** Early in design discussion, map which inputs/outputs the
changes share. If they overlap: keep ADRs as separate documents with individual
approval, but implement under one version bump and mark the release notes
"together".

## Documentation

### Architecture docs explain "why"

**Symptom:** Explicit feedback that structure-only descriptions don't carry the
context needed for future change decisions.

**How to apply:**
- Structure sections as "why X", including **rejected alternatives and their
  reasons**.
- Ground claims in operational evidence (measurements, incidents).
- Don't write "what" that code already answers.
- For large innovations, split per subsystem and link from an overview document.

### Docs for prompt-driven tools include the original prompt verbatim

**Symptom:** Documenting prompt content only in translation creates drift from
what is actually sent to the LLM and misleads readers.

**How to apply:** Quote the original (English) prompt verbatim in a blockquote
with the translation below. Specify evaluation criteria (severity etc.) as
tables with examples. Common boilerplate: full text at first occurrence,
elision afterwards.

### Attribute even "inspired-by" sources

**Symptom:** Even when code is informed by (not ported from) another project,
attribution costs zero while omission risk is non-zero.

**How to apply:** Whenever a design or heuristic is informed by another project,
regardless of "derivative work" status: (1) reference project/author/license in a
source-file comment; (2) add a Third-Party Notices section to LICENSE.

## GitHub operations

### Repositories are no longer watched automatically (auto-watch sunset 2025-05)

**Symptom:** Watch states across 124 org repositories had drifted — about half
the active repos sat on Default, meaning **external issues/PRs produced no
notifications**. The cause: GitHub sunset "Automatically watch repositories" on
2025-05-23. Repos created before the sunset had been auto-watched; everything
created after stayed on Default — a fault line by creation date. This is also
why the toggle is gone from the settings page.

**Why it matters:** Default (Participating & @mentions) does not notify even the
owner of new issues/PRs. In a solo-run org, external contributions to unwatched
repos sink silently.

**How to apply:**
- Watch explicitly right after creating a repository:
  `gh api -X PUT /repos/<org>/<name>/subscription -F subscribed=true`
  (part of the scaffold checklist).
- On archiving, switch to Ignore (archived repos accept no issues/PRs, so a
  lingering watch is pure noise): `-F ignored=true`.
- Auditing the current state is mechanical via the REST API:
  `GET /repos/<org>/<name>/subscription` (404 = Default, `subscribed` =
  Watching, `ignored` = Ignoring).

### `(#N)` is reference-only — auto-close needs a Closes keyword

**Symptom:** Committing `fix(...): ... (#15)` did not close the issue; it stayed
open and was pointed out. Releasing via direct pushes to main (no PRs) means
nothing closes issues unless the commit body says so.

**How to apply:** Issue-resolving commits add `Closes #N` at the end of the body
in addition to the parenthesized reference. Multiple issues need a keyword per
number (`Closes #15, closes #16` — GitHub ignores extra numbers after one
Closes). Put it on the fix commit, not the release commit.
