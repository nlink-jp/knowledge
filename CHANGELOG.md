# Changelog

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
