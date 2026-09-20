# Config, Identifiers & CLI IO

Config-file loading, identifier canonicalization, the filesystem, OAuth, and
terminal IO. Each entry follows **symptom → why → how to apply**.

---

## Identifiers & data formats

### Fix wire-format mismatches with BOTH source canonicalization and boundary normalization

**Symptom:** The format of tool names sent to the LLM (kebab vs snake) was
rejected by one backend, failing dispatch. "Boundary normalization only" (source
stays kebab) was rejected — the source would lie about what's on the wire.
"Source rename only" breaks old session history, scaffolded user data, and
external servers' published names.

**How to apply:**
- **What you control (source)**: rename spellings in code to the canonical form
  matching the wire. A source that lies permanently raises reader cognitive
  load.
- **What you don't control (external input)**: apply a single `canonicalize()`
  helper at registry boundaries (lookup / dispatch ingress / schema emit /
  persisted-config load) to pass old data through without migration.
- Either alone is insufficient. "Dispatcher accepts aliases" leaves the schema
  lying — long-term debt.

### When canonical forms change, migrate user config files at load time

**Symptom:** Canonicalizing only the lookup side left previously saved user
settings (override maps / disabled lists keyed by the old form) **silently
ignored**. Users hit "I remember toggling that and it isn't working".

**How to apply:**
- When planning an identifier change, grep for **every place that persists maps/
  lists keyed by it** (config, session metadata, export/import formats…).
- Minimal migration: **once at load, in-place rewrite** (the next save writes the
  canonical form back). Degenerate old+new-key coexistence resolves canonical-
  wins.
- Caveat: tests that inject values into constructed objects bypass the migration
  path — test both input forms.

### Never give cross-machine portable data machine-local IDs

**Symptom:** Exported/imported data referenced timestamp-derived session IDs
(`sess-<unixMilli>`). Machines share the wall-clock millisecond space, so
collisions across machines are near-certain, turning any future use of those
references into a "acts on someone else's session" bug. The fields were also
write-only dead weight.

**How to apply:**
- **YAGNI**: delete provenance fields with zero reads (verify per-field with grep
  for real set/read).
- If a field must stay, **unconditionally clear foreign IDs at the import
  boundary** (matching against local lists creates false links — worse).
- Wrap exports in a `kind` + `schema_version` envelope to fail fast on wrong
  files. Default imports to merge + duplicate-skip (non-destructive,
  idempotent).
- Machine-generated JSON read with plain Unmarshal (non-strict) ignores old keys,
  which disappear on next save = no migration needed.

### An id-ordered listing is chronological only while ids are timestamps — carry time in its own column

**Symptom:** A tool aggregating session transcripts listed sessions in id-string
order. While ids were `YYYYMMDD-HHMMSS` that looked chronological; from the
producer version that switched ids to UUID v4, new sessions sorted after every
old id in hex order and a row no longer said when it ran. The parser never
depended on the file name, so ingest kept working and only the display broke —
which is why every test stayed green and nobody noticed.

**Why:** "sorting by id gives time order" is an implicit dependency on the
content of the id, written nowhere. The producer defined compatibility as
"existing files still load and resume" (true), and the consumer's ordering is
outside that contract. An id-format change is the kind a producer's ADR calls
"compatible, ships as a minor" — the consumer has to go and check its own
follow-through.

**How to apply:**
- A listing meant to be read in time order **carries time separately from the
  id** (first / last record time on each aggregated row, `--sort time` as the
  default). Treat the id as opaque.
- When the producer publishes an ADR about "session" or "id", check the
  consumer's **display, ordering and key width** on real data. A green parser
  test says nothing about the table.
- Write the added time keys **always (`""` when empty, no `omitempty`)** so
  that key absence means only "an older CLI" — a GUI that starts reading them
  later can tell the cases apart (the same reason as `tool_prompt` in
  llm-integration).
- Do not apply a time sort to a dense series with filler rows (`--dense`): a
  row with no record has no time, sinks to the end, and breaks a series that
  was already correct by key. Refusing the combination is safer than adding a
  fallback to the comparator (transitivity survives).

### Long-format storage makes re-importing idempotent

**Symptom:** history from an external service has to be imported after the fact to
fill gaps. Export ranges overlap easily, inviting double insertion or silent loss.

**Why:** with "one row per record, whose primary key *is* the record's identity",
`INSERT OR IGNORE` becomes exactly "add only what is missing". Store the same data
as several columns on one row and the application has to work out partial overlap
for itself.

**How to apply:**
- Model time series as long format: `(subject, metric, timestamp) → value`. The
  primary key supplies idempotency for free.
- A side benefit: **unknown fields land without a schema change**. The importer
  does not break when the external service starts reporting something new.
- Give the import command a `--dry-run` reporting how many rows are new versus
  already held. Implement it by doing the work in a transaction and rolling back
  — an exact count rather than an estimate.

### macOS hands an unprivileged process altered per-interface byte counters — a field's width does not guarantee the value's width

**Symptom:** to show per-interface send and receive rates, the cumulative byte counters in
`if_data64` were read through `sysctl` (`NET_RT_IFLIST2`). The fields are 64 bits wide, so
the design began from "a wrap cannot happen" — until the values were compared with the OS's
own `netstat -ib` at the same moment and did not match. What the home-made process received
was **the true value modulo 2^32, then floored to 1 KiB**. At 9 MB/s that wraps about every
8 minutes. (2026-09. On a macOS 27.0 machine, wired: two comparisons bracketed by `netstat`
readings, run independently. On a macOS 26.6.2 VM the 1 KiB flooring was confirmed once;
32-bit truncation there is unconfirmed, because the counters were below 4 GiB and it could
not be told.)

**Why:** a type's width says which values can fit, not which values are actually handed
over. An OS may return different precision from the same API depending on the caller. Why
the bundled tool and the home-made process differed was not established (a privilege only
the bundled tool holds is the guess). Neither the compiler nor a test finds this kind of
alteration: the numbers look plausible, and so do the rates computed from their deltas. It
only shows at the moment of a wrap, as a spike of several GB/s.

**How to apply:**
- When a design reads an OS counter, **compare it with an unaltered reference before
  trusting the type**. Bracket it — reference, own reading, reference — and check that the
  own reading falls between the two references taken modulo 2^32. A one-sided comparison
  cannot be told apart from the traffic that flowed in between.
- Truncation cannot be observed until the true counter has passed 2^32. "They matched" on a
  freshly booted machine or a VM is no evidence that there is no truncation.
- **Always take deltas modulo 2^32.** 1024 divides 2^32, so flooring and the modulus
  commute; the result is correct on an OS that truncates and on one that does not, as long
  as one sample's increase stays below 4 GiB.
- That condition depends on **a bounded sampling interval**. A sample stretched by a sleep
  or a stalled timer is discarded and the baseline re-established, whatever the values say.
  Measure elapsed time with a clock that keeps running during sleep; with one that stops, a
  sleep hides inside an interval of ordinary length.
- Treat a wrap and a reset (an interface being re-created, for instance) as different
  things. A link-speed cap alone does not separate them: if the delta after a reset is
  spread evenly over 4 GiB, a 10GbE cap accepts about three in ten as a normal increase.
- The resolution is 1 KiB per sample. Traffic below that alternates between 0 and one unit.
- When reading `netstat -ibn` mechanically, a row without an address (loopback, tunnels) has
  one column fewer. A fixed column number reads the packet count as bytes on those rows only.

## Data retention

### A retention period is not a retention depth — ingestion lag eats into what you keep

**Symptom:** Intending to keep 400 days, a partition expiration of 400 days was configured; only
399 days were actually queryable. On top of that, of the 450 days of history the source backfilled,
the oldest 50 days vanished as they landed. **Neither produced an error or a warning.**

**Why:** The retention window is bounded at both ends. The old end is "today − expiration", which
the lifecycle setting deletes. The new end is "today − ingestion lag", which the source has not
published yet. Expiration only moves the old end, so:

```
days actually retained = expiration − ingestion lag + 1
```

For data that arrives 1-3 days late, such as daily rollups, you are always short by the lag. And
when a bulk historical load (backfill) lands in a destination that already has an expiration, the
**oldest partitions expire as fast as they arrive**. Both the load and the deletion are behaving
correctly, so nothing surfaces as an anomaly.

**How to apply:**

- When the retention depth is a requirement, set the expiration to **"days you want + ingestion
  lag"**. If the lag varies, budget for the worst case.
- **If the oldest surviving day equals "today − expiration" exactly, you are deleting it yourself,
  not hitting the source's limit.** A source-side limit leaves the oldest day on a date unrelated to
  your setting, and it does not advance as days pass. Your own expiration makes it advance by one
  day, every day. That is how you tell them apart.
- **Do not set an expiration until the backfill has finished** (or set it above the maximum the
  source can deliver, then wait). A load can take days, and its intermediate state looks sparse and
  broken. Raising the expiration afterwards does not resurrect what was deleted.
- **A default expiration only applies to objects created after it is set.** The one clean moment is
  right after creating the container and before any content exists. Miss it and you must enumerate
  the existing objects and set each one — anything you miss grows without limit.
- **Do not govern a container holding data with different lags by a single default.** When
  per-event data (lag in minutes) and daily rollups (lag in days) share a container, only the
  latter comes up short.
- An expiration promises "delete anything older than this", not "keep this much". It rolls: one day
  drops off the tail every day, and depth never accumulates. Where long-term retention is an
  obligation, provision a separate tier that is never deleted, alongside the expiring hot tier. The
  hot tier's expiration is a setting about query speed and cost, not about preserving evidence.

Origin: retention design for an audit-log export into a data warehouse (2026-09).

## Config files

### Read hand-written JSON/TOML with strict decoding

**Symptom:** A user's config contained a field-name typo (`callbackSchema` for
`callbackScheme`); default Unmarshal silently ignored it and fell back to the
default value. "I specified HTTPS but it comes as http" took real time to trace.

**How to apply:** For user-authored config: Go `json.Decoder` +
`DisallowUnknownFields()`, Python pydantic `ConfigDict(extra="forbid")`. Add one
test that a typo field is rejected.
**Machine-generated data is the opposite — plain Unmarshal** (old fields must be
ignored for backward compatibility; see the machine-local-ID entry). The
distinction matters.

### Changing a "where to store" setting does not move already-persisted absolute paths

**Symptom:** Installed asset files were moved to another volume and the setting
that selects the storage directory (`models_dir` / `output_dir` / `cache_dir` …)
was repointed at the new location. The **registry / index / manifest recording an
absolute path per file** kept the old one, leaving every existing entry aimed at
a directory that no longer exists.

**Why:** A storage-directory setting only decides where files will be written
**from now on**; it never touches records already written. The two are separate
facts, even though the user sees a single "where my models live".

**How to apply:**
- The moment you add a "change the storage directory" setting, design "how do
  existing entries move" with it. Document the migration in the README as three
  steps: move the files → change the setting → run reconcile.
- **Ship a reconcile command alongside it.** Dry-run by default, `--apply` to
  write. Rewrite a path **iff** the recorded path does **not** exist **and** a
  file of the same basename **does** exist in the new directory. Never touch a
  path that still resolves (a user deliberately spreading assets across
  directories keeps them). Report entries with no candidate as unresolved —
  never guess.
- A `--to DIR` override of the target directory makes the command **its own
  undo**: re-run it pointing at the old directory.
- Reconcile rewrites records only — it **never moves bytes** (that is the user's
  `mv`). Dry-running before the files move correctly reports everything as
  unresolved.
- **Do not rebase implicitly at load time.** Resolving by basename can silently
  grab an unrelated same-named file, and the on-disk records keep disagreeing
  with what the process actually used. State that repairs itself invisibly is
  harder to reason about than state that reports its own breakage — prefer an
  explicit command plus honest listings.
- Centralize enumeration of the recorded file fields in one place (a
  `fileRefs()`-style accessor returning get/set per field). If listing,
  existence-checking, and rewriting each walk the struct separately, adding a
  field will be honored by one caller and missed by another. Inject the
  existence check as a function and all of it tests against a synthetic FS.

### Listings, pickers, and catalogs must verify that referenced files exist

**Symptom:** The installed-items listing read the registry and **never stat'd the
files**, so every entry showed as installed and healthy even with the referenced
files absent. The CLI table, `--json`, the MCP tool, and the GUI management
window all shared the same unverified view, so the first code to open a file was
the first to fail — with a stale path. The user-visible symptom ("the manager
sees my items, but using one fails") points nowhere near the cause.

**Why:** Unverified state was presented as fact. No layer checked that the source
of the display (the records) and the truth (the disk) still agree. Missing path
reconciliation and missing existence checks are **two independent defects**;
together they postpone discovery until run time.

**How to apply:**
- Stat the referenced files before returning a listing. Surface absence in the
  row (a `STATUS` column, plus a footer naming the missing files and the command
  that fixes them). An unmounted external volume is the same shape — the files
  are not gone, but the listing must not claim they are there.
- Return a **list of missing paths** (`missing_files`) to front-ends, not a
  bool — it lets them explain the cause. Omit when empty, and **treat a missing
  key as healthy** so older clients keep working.
- **Front-ends keep unloadable entries out of the picker but leave them in the
  management view** (invisible means unfixable and undeletable), with text
  explaining why they vanished. Anything shown as available must be loadable.
- Reject early in the runtime resolution layer too, naming the missing file, the
  current setting, and the remedy command. Never surface the underlying
  library's low-level error as-is.
- Build listings from a **single shared view** and the CLI, JSON, MCP, and GUI
  are all fixed at once; per-surface views let the check be forgotten again.
- Cost is O(files) stats per invocation. Missing or unmounted paths fail
  immediately with ENOENT, so the degenerate case is the fast one.

### On macOS, search both Application Support and ~/.config for the config file

**Symptom:** a user put their credentials in `~/.config/<app>/config.toml` while
the app looked only in `~/Library/Application Support/<app>/`, and it went on
reporting "no credentials".

**Why:** two conventions coexist on macOS, and CLI users reach for `~/.config`
there as everywhere else. **A config file that exists and is silently never read
is far worse than one reported missing** — its existence convinces the user the
setup is done.

**How to apply:**
- Read from a search list in priority order: `XDG_CONFIG_HOME/<app>` →
  `~/.config/<app>` → `~/Library/Application Support/<app>`; first hit wins.
- The canonical *write* location can stay Application Support. Only the config
  path forks — keep the data directory (database and so on) single.
- **Have the diagnostic command print which file it read, and list the paths it
  searched when there is none.** That is the one thing that makes this class of
  accident diagnosable at a glance.
- Tests must override **both** `HOME` and `XDG_CONFIG_HOME`; overriding one lets
  the developer's own real config leak in and break the test.

### A persisted verdict needs a path to be re-decided

**Symptom:** whether a device was in scope was probed once and stored as a
boolean. Correcting the classification logic **never reached the records already
classified**.

**Why:** it is the same failure as neglecting to migrate persisted settings, but
harder to notice, because the stored value was derived by us rather than entered
by the user — so fixing the logic feels like fixing the data.

**How to apply:** when persisting a derived verdict, always ship an explicit
re-decide command or control (`--reclassify` and the like), and document it as
the only migration path. If re-deciding is expensive (API calls, say), keep it
off the default path and make it deliberate.

### Reverting someone's settings: record what you observed, never what you intended

**Symptom:** A tool that changes another program's settings and offers to put
them back produces four failures that all look unrelated — it refuses to restore
a value the user had set by hand; it deletes a setting the user had set; after a
write that only half landed it accuses "something else" of changing the settings
and refuses to clean up; and it silently overwrites a change someone made
outside it. In menubar-spacer (macOS status-item spacing preferences, 2026-09)
an independent review found all four in one write layer.

**Why:** They are one defect. The backup record was built from what the tool
*meant* to write, while the read layer quietly reshaped what it *saw*:

- The record's "what we applied" field held the requested target. A write the OS
  honoured for one key only left the record describing a state the machine was
  not in, so the restore path — which compares the live state against that field
  to detect outside interference — classified the tool's own half-write as
  someone else's change.
- A plausibility range (0…64) was applied to values *read from the machine*. A
  hand-set 100 was captured faithfully and then rejected as corrupt by the very
  check meant to guard against corruption, leaving no in-app way back.
- Values were coerced to the type the tool understands. `defaults write -g
  SomeKey 8` without `-int` stores a **string**; read as an integer it came back
  as "not set", so a restore would delete a key the user had set. A float was
  truncated.

**How to apply:**
- Fill every field of the record from a read-back, never from the request. After
  writing, re-read and re-save; the record must describe the machine even when
  the write went wrong.
- Validate a record when you **decode a file**. Never validate a value you
  **observed on the machine** — a range check there refuses exactly the users
  whose settings are furthest from default, which is who needs restoring most.
- Keep a value you cannot interpret verbatim (a property list blob, a raw byte
  string) and write it back unchanged. Model it as its own case so nothing can
  confuse "a value I don't understand" with "no value".
- Treat a state where each key holds either the original or the last observed
  value as yours to clean up. Anything else is outside interference — and then
  undo refuses, while an explicit new request proceeds but reports what it
  replaced. Refusing an explicit request is obstruction; undoing over an
  unexplained state is destruction.
- Store the way back before the first write, hold an exclusive lock across
  read → decide → save → write, and read the live state inside that lock. Two
  copies of the tool otherwise record a state the tool itself produced as the
  user's original.
- Move an unreadable record aside instead of deleting it, and only after a write
  actually happened. The read may have failed transiently, and its bytes may
  still be readable by a person.

### A key that is present is a value that was given — do not read `""`, NaN or a sentinel as "not set"

**Symptom:** One tool's independent review turned up the same kind of defect in three places.
(1) `base_url = ""` in the config file was skipped, line and all, by a loader written as
`if v != ""`, and the default URL was used without a word. (2) TTLs and the timeout were read
with `strconv.ParseFloat` and refused on `x <= 0`; `NaN` makes every comparison false and went
through, and `Inf` or `1e300` overflowed `time.Duration`. (3) The CLI's `--min-cvss` used a
default of `-1` as its "not given" sentinel, so `--min-cvss -1` was not refused as out of
range, and an unfiltered list came back as if it had been filtered.

**Why:** All three infer from a value's content whether it was given. Whether it was given is
a matter of **presence**, not of value. When the inference is wrong, what happens is not a
refusal but silence, and the user believes the setting took effect — the same defect as a
config loader that silently ignores unknown keys.

**How to apply:**
- Read config file keys **by presence** (`if v, ok := section[key]; ok`). An empty string goes
  on to the validator and is refused by name. Only an environment variable may read "" as
  unset (emptying one is the usual way to clear it) — say so in a comment.
- Accept a number **inside its range**: `if !(f > 0 && f <= max)`. The negated `f <= 0` lets
  NaN through. Derive the upper bound from what the type can hold.
- Ask `flag.Visit` whether a CLI flag was given. Do not compare against a sentinel: a
  sentinel is a value the user can type.
- When a limit in the config and a limit in the engine are one fact — a page ceiling — and
  neither package can import the other, tie them in a test that imports both. Clamp a
  configured default that exceeds a tool's ceiling; refuse only a value the caller sent.

### Do not build a cache key by joining parts with a separator — hash length-prefixed parts, and do not fold case

**Symptom:** A result cache built its keys as `kind + "_" + vendor + "_" + product`. The tokens
may contain `_`, so vendor `a` + product `b_c` shared a slot with vendor `a_b` + product `c`,
and the second query was answered with the first one's result. Lower-casing the key to make it
a safe file name merged identifiers that differ only in case (vendor advisory IDs mix case).

**Why:** Joining is injective only while no part contains the separator. How the character set
the input gate allows relates to the separator breaks silently the day the gate is loosened.

**How to apply:**
- Build keys in one function only. Feed each part to a hash **length-prefixed**
  (`len:part|`), and use `kind` plus the head of the digest as the file name. Never join by
  hand.
- Normalize a part (lower-case it, say) only where you have **confirmed** that upstream treats
  that identifier case-insensitively.
- Keep a test of the pairs that would collide: a split across the separator, an empty part, a
  difference in case only, another `kind`. Check through the engine as well that the second
  query reaches upstream.

## Filesystem

### Never read volume free space from volumeAvailableCapacityForImportantUsage alone

**Symptom:** a macOS app that runs a pre-flight free-space budget check before
extraction (ZIP-bomb defence) refused **every** extraction onto an SMB share
with "insufficient space — required: 414 MB / free: 0 KB". The server had over
3 TB free.

**Why:** `volumeAvailableCapacityForImportantUsage` (the purgeable-aware
"effective free space") only answers for local APFS volumes; on network mounts
(SMB/NFS) it returns **0, not nil**. Because a value "came back", taking it at
face value misreads 0 as a full disk. The statfs-backed
`volumeAvailableCapacity` reports the real figure on network filesystems.

**How to apply:**
- Accept the important-usage key **only when it is positive**; treat 0 or
  negative as "no answer" and fall back to `volumeAvailableCapacity`. When both
  answer, take the larger (on local APFS that is the purgeable-aware one).
- A genuinely full disk still refuses after the fallback — statfs reports ~0
  there too, so the safety property survives.
- When neither answers, skip the check rather than fabricating a figure.
- Extract the decision into a pure function and unit-test it
  (`(0, 3TB) → 3TB`, `(0, 0) → 0`, `(nil, nil) → nil`).

### Replace-by-rename drops the file mode — leaving it to the call site is how one call site gets it wrong

**Symptom:** A helper that replaced a file safely (write a temp file, rename it into place) took
the new file's permissions as a parameter. It had two callers: one passed the stat'd mode of the
existing file and was right, the other passed a literal `0644`. Through the second, every
overwrite dropped the execute bit off scripts and widened `0600` files to `0644`.

**Why:** Rename installs a new inode, so the mode is lost unless it is carried across
deliberately. Making that the caller's job turns it into a convention — "pass the right mode
every time" — and conventions break.

**How to apply:** Have the replacing function itself lstat the name it is replacing and inherit
that mode. Demote the parameter to "the mode for a file that does not exist yet", or remove it.
Apply a preserved mode with `chmod` on the open descriptor, since `OpenFile`'s mode is masked by
umask. But **respect umask for a file that did not exist** — forcing the default past it widens
files under a restrictive umask, which is the same defect pointed the other way. Inherit only
the mode of the name being replaced, never a symlink target's.

## OAuth

### Treat refresh-less tokens as non-expiring — never fabricate an expiry

**Symptom:** When the token exchange returned no `expires_in`, the code invented
"now + 1 hour". The provider (Slack with token rotation disabled) actually issues
non-expiring tokens, so after an hour the "expired + no refresh" state killed a
perfectly valid token and forced hourly re-login.

**How to apply:**
- No `expires_in` + no `refresh_token` → store `expires_at = 0` (unknown) and
  use the token as-is. Detect real revocation via upstream **401** and error
  there.
- Only with no `expires_in` + a `refresh_token` present is a probe value ("check
  again in an hour") reasonable.
- "`expires_at` exactly save-time + 1 h" is the tell of a fabricated expiry.

### Some providers reject http:// loopback redirect_uris

**Symptom:** RFC 8252 recommends `http://127.0.0.1:<port>/callback`, but some
providers' app-registration UIs flatly reject non-`https://` URIs (not a spec
violation — the RFC binds clients, not providers).

**How to apply:** Provide an HTTPS loopback path:
1. Generate an ephemeral self-signed cert (IP SANs 127.0.0.1 + ::1, DNS SAN
   localhost, ECDSA P-256, short validity, memory-only).
2. Wrap the TCP listener with `tls.NewListener`.
3. Use `https://localhost:<port>/callback` (some UIs also reject IP literals —
   use the DNS name).
4. Tell users the browser warning is expected — one click-through.

## Terminal & input UI

### Multi-line pastes leak into the shell under single-line reads

**Symptom:** An interactive prompt read one line via `bufio.Scanner`; pasting
multi-line text leaked the remaining lines to the shell, which **started
executing them as commands**.

**How to apply:**
- Use a multi-line read mode terminated by an empty line for interactive input.
- Offer `--input` to read long input from a file.
- Grow the Scanner buffer beyond the 64 KB default.

### Input-history navigation requires an "active" flag

**Symptom:** With ArrowUp/Down history navigation, pressing ArrowDown at
line-end during normal typing misfired the history logic and overwrote the input
with the (empty) draft — **the user's in-progress input vanished**.

**How to apply:**
- First line of the ArrowDown handler: return unless history navigation is
  active.
- Consume keypresses only while navigating; otherwise defer to the default
  behavior (cursor movement etc.).
- Generalization: any UI that "steals keys only in a particular state" (history,
  autocomplete, IME) must check the state flag first and pass everything else
  through.

### Fix mixed-language UI with a complete two-sided catalog + completeness test, not translation

**Symptom:** a CLI's interactive chrome (help, hints, dialogs) drifted into a
Japanese/English mix. Each string was added in whichever language the moment
suggested; none is individually wrong. Fixing the mix by hand does not last —
the next feature adds its strings in one language again. The problem is not a
bad translation but the absence of anything forcing both languages to cover
the same surface.

**How to apply:**
- Define ONE `Messages` struct — one field per string — with two complete
  per-language literals (gem-agent `internal/uitext`).
- **Make a reflection test fail on any empty field in either catalog.** That
  test is the actual fix: a new string either lands in both languages or
  fails CI.
- For fields containing `%d`/`%s`, also test that the fmt verbs match in
  order across catalogs (a mismatch compiles fine and breaks one language at
  runtime only).
- Resolve the language POSIX-style (first non-empty of `LC_ALL` →
  `LC_MESSAGES` → `LANG`, a `ja` prefix selects Japanese, `C`/`POSIX` mean
  English), once at startup — a mid-session switch bisects the scrollback.
- **Declare the out-of-scope surfaces up front**: grep-stable log-shaped
  lines, `--help` (CLI convention), model-facing text, and library error
  chains stay English. Translating everything puts a localized prefix on an
  English chain — the very mixing being removed.

### Marker/flag files read by other processes must land by temp+rename (os.WriteFile is create-then-write)

**Symptom:** a `.project` ownership marker written with `os.WriteFile` made
roughly half of all parallel same-project launches refuse startup as a
"path-escape collision". WriteFile opens O_CREATE|O_TRUNC — the file exists
EMPTY before the content lands — and a concurrent reader in that window sees
"owner = (empty)".

**How to apply:**
- Any marker/flag/small-state file another process may read gets written to a
  temp file in the same directory, then `os.Rename`d into place. Rename is
  atomic: readers see no file or the whole file — keep both states safely
  interpretable.
- **Treat an empty file as "unowned", same as absent, and let the next writer
  repair it** (tolerance for pre-fix crash leftovers).
- A residual check-then-write TOCTOU is acceptable when the file is a
  best-effort disambiguator rather than a lock — last rename wins; say so in
  a comment.

### Cloud-storage writers COMMIT on Close — abort by cancelling the context

**Symptom:** when `io.Copy` into a GCS upload failed mid-stream, the
cleanup-looking `w.Close()` **finalized the buffered partial data as the
complete object**. In a content-addressed store (hash names, nothing ever
deleted) the broken content now lives permanently under the correct name,
and dedupe serves it forever.

**How to apply:**
- For `cloud.google.com/go/storage`, Close means commit; an error-path Close
  is a partial commit, not a cleanup. **The documented abort is cancelling
  the context passed to NewWriter** — derive a per-writer
  `context.WithCancel`, cancel on copy error, then Close.
- Defend the name=content invariant itself: hash and upload from ONE file
  descriptor (re-opening by path loses to a rename-replace), and re-hash the
  upload stream with a verifying reader that fails before EOF if the file
  changed mid-upload. Design permanent stores as if a wrong entry can never
  be fixed later — because here it can't.

### exec cancellation kills only the DIRECT child — a pipe-holding grandchild blocks Wait forever

**Symptom:** an agent's shell tool (sandbox-exec → bash → python) hung
forever through both its timeout and the operator's Ctrl+C.
`exec.CommandContext` kills the direct child only; the python grandchild
survives holding the inherited stdout/stderr pipe, and `CombinedOutput`'s
internal `Wait` blocks until that pipe reaches EOF — **the timeout fired
and was then defeated**, and the later cancel wedged in the same Wait.
"The tool never responds" and "Ctrl+C does nothing" were two symptoms of
one hole.

**How to apply:**
- Any command that may spawn children gets all three: ① `SysProcAttr.
  Setpgid = true` (child leads a process group), ② `cmd.Cancel` sending
  SIGKILL **to the group** (`syscall.Kill(-pid, SIGKILL)`), ③ a
  `cmd.WaitDelay` of a few seconds — even a setsid/double-fork escapee
  cannot keep Wait from returning. The kill is best-effort; **the return
  is guaranteed** (one orphan is far cheaper than a hung session).
- Reproduce with `bash -c 'sleep 30 & sleep 30'` (a background child
  inheriting the pipe) plus a short timeout/cancel and a few-second
  watchdog: the unfixed code hangs every time.
- Add UI defense in depth for tools that ignore cancellation anyway: an
  escape ladder (second Ctrl+C warns, third quits) — with an append-only
  record you can honestly say "everything up to here is saved".

### In-process walks need the same "return is guaranteed" contract — a tool that merely receives ctx ignores Ctrl+C

**Symptom:** Ctrl+C during an agent's file search stayed on
"interrupting…" while the walk ran on; on a slow filesystem the wait
was the whole remaining walk. `search_files`/`list_tree` took `ctx` as
a parameter and never consulted `ctx.Err()`, and the caller ran
`tool.Run(ctx, …)` synchronously with no floor under it — all Ctrl+C
did was cancel a context nobody read. Fixing the exec grandchild hole
(the entry above) for the shell had given the in-process tools no
equivalent contract. A faithful probe (30k files, SSD) cancelled at
20 ms: the shipped walk returned after 1.6 s (the full walk), a walk
with per-dir/per-file ctx checks after 2 ms, a goroutine+select
wrapper after 1 ms.

**How to apply:**
- Two layers. ① **Cooperative**: the walk checks `ctx.Err()` before
  every directory and file read (and every N lines of a long file) and
  returns what it found, labelled "[interrupted after N files —
  partial]" — a partial result is a result, not an error, and a cut is
  never silent. ② **The floor**: the caller runs `Run` in a goroutine,
  selects on `ctx.Done()`, waits a short grace (~1 s) for the
  cooperative return after a cancel, then abandons the call — a
  blocking syscall (ReadDir on a hung NFS/SMB mount) cannot be stopped
  from Go, so ① alone is not enough. Stop is best-effort, return is
  guaranteed: the exec rule again.
- Size the grace so the cooperative return wins the race: one syscall
  on a healthy filesystem is short, but keep it **longer than exec's
  `WaitDelay`** or the floor discards a shell call's output a moment
  before Wait returns it. Pin the ordering with a test.
- Keep outside the floor: the approval gate, and any **tool that waits
  on the operator's own input** (ask_user). An abandoned stdin read is
  a second reader on the shared stdin and silently eats the next typed
  line. Let the tool declare it with a flag (WaitsOnOperator).
- Account for what you abandon: count running abandoned calls on the
  exit receipt, write an audit record on a late return, and when a
  **mutating** call completes late, tell the model at the start of the
  next turn (its last word was "result discarded"; the write landed).
- The escape ladder belongs at every entry point: `signal.NotifyContext`
  swallows every SIGINT after the first while registered, so a
  three-press exit built only into the TUI leaves the plain REPL/-p
  with no way out. Pin the call sites with a source-scan test.
- The reproduction can be made deterministic with a FIFO: mkfifo in the
  walk's first directory; the test's open-for-write blocks until the
  walk is inside the read (the sync point), then cancel, then close for
  EOF. Unfixed code walks on into the next directory; fixed code
  returns at its next check.

### One width model per Go TUI — go-runewidth treats Ambiguous glyphs as wide under a CJK locale

**Symptom:** In a Bubble Tea + glamour TUI, box art placed in a code
block (an ER diagram) broke in a telling shape: **only lines containing
text had their box gap widened; lines made of box-drawing characters
only were fine** — and the scrollback hard-wrap then sheared the
over-long tails. Under `LANG=ja_JP.UTF-8` go-runewidth sets
`EastAsianWidth=true` and counts East Asian Ambiguous glyphs — box
drawing ─│┌┐├┤, arrows ►◄, "…" — as two cells, while x/ansi, uniseg, and
the common terminal setting in the same process count one. glamour pads
each code-block line with go-runewidth, so lines with many box
characters looked "already wide" and got less padding; real widths
varied per line (measured 176/125/172/125 cells on consecutive lines).
Under `LANG=C` every line was 176.

**How to apply:**
1. **Pin every width measurer in the process to one model**: at startup
   set `runewidth.DefaultCondition.EastAsianWidth = false` (agreeing
   with x/ansi, uniseg, and most terminals) — but honour an explicit
   `RUNEWIDTH_EASTASIAN`, leaving an opt-in for operators whose
   terminal renders Ambiguous glyphs wide.
2. When a new width-measuring dependency arrives, **test it with
   `EastAsianWidth = true` forced** and assert that rendered lines
   containing box characters all have equal real width — a
   locale-dependent bug does not reproduce unless the test reproduces
   the locale on purpose.
3. Telling the two failure modes apart: "only text-bearing lines
   drift, box-only lines are fine" → a width-measurement mismatch in
   the program; "only box-bearing lines drift" → the terminal's own
   ambiguous-width setting.

### Call Focus() on a bubbles component BEFORE storing it into the Bubble Tea model

**Symptom:** A one-line textinput field added to a TUI approval dialog
ignored every keystroke — placeholder and cursor rendered, not a single
character landed. The wiring looked correct; only the unit test's
Value() assertion failed (Go CLI agent, 2026-08).

**Why:** Bubble Tea models travel by value copy through every Update.
Written as `ti := textinput.New()` → configure → `m.input = ti` →
`ti.Focus()`, Focus (a pointer receiver) mutates only the local
variable; the copy already stored in the model stays unfocused. An
unfocused textinput.Update silently drops keys, so there is no error
and no panic — just a field that cannot be typed into. Same root cause
as the noCopy-type panics (strings.Builder et al.), but this variant
makes no noise, which is worse.

**How to apply:**
1. Fix the order: create → configure → **Focus() (keep the returned
   tea.Cmd) → assign into the model last**. Cursor blink starts from
   that tea.Cmd, so return it rather than dropping it.
2. Every time a bubbles component is added to a model, ask "did every
   pointer-receiver call happen before the copy?" — an addition to the
   existing "does this type survive value copies?" check.
3. Pin it with a regression test that feeds a KeyMsg AFTER the field
   is stored in the model and asserts Value() changed. View-level
   checks (the field renders) cannot catch it.

### Calling Bubble Tea's `Program.Send` from inside Update freezes the whole UI — and the callbacks that run inside Update do not look like it

**Symptom:** The TUI stopped accepting input entirely; neither Ctrl+C nor
Ctrl+D ended it, and it had to be killed from outside. **The same mechanism
has done this twice** — first through a slash command (`/clear`) that ran a
hook which reached `prog.Send`, then through another slash command that sent
an image to the screen (2026-09, an agent's TUI).

**Why:** Bubble Tea v1's `msgs` channel is unbuffered, and the only reader is
the goroutine running Update. A `Send` from inside Update's call stack waits
for something nobody will read until Update returns. Later keystrokes queue
on the same channel, so in raw mode even Ctrl+C is just another key that
never arrives.

What makes it hard to see is that **the callbacks running inside Update do
not look like it**. Slash handlers, applying a settings change, running a
hook — each is written as "a layer outside the UI" and each is in fact called
synchronously from `Update`. The same function works at startup (no program
yet) and mid-turn (another goroutine), so reading the code suggests Send is
fine.

**How to apply:**
- Do not hand the Program to a callback that can be called synchronously from
  Update. Return the notification as the callback's **return value** — folded
  into the slash output, where its ordering is deterministic too.
- If it must be sent, use `go prog.Send(...)`. That loses ordering, so do not
  claim delivery in a return value; what a caller can act on is whether there
  is a program at all.
- Review heuristic: **enumerate every caller of `prog.Send`** and ask of each
  "can this run synchronously from Update?". One yes is a freeze.
- Pin it with a sender that never returns, and assert the caller returns
  anyway. The synchronous version fails that on a timeout. Unit tests stay
  green while the UI dies, so without this shape the next one is found on real
  hardware.



**Symptom:** A subcommand rendered its formatted report with
`printReport(fs.Output(), …)`. The `--json` variant went to stdout, the text
variant went to stderr (a usage-accounting CLI, 2026-09). Unit tests injected
an `io.Writer`, so they stayed green. An independent verifier ran the command
with `2>/dev/null`, saw nothing, and caught it.

**Why:** `flag.FlagSet.Output()` is the writer for usage and flag errors and
defaults to **os.Stderr**. "The writer the FlagSet already holds" looks like a
handy destination, but it serves a different purpose. When tests inject the
writer, the one production call site is the one thing they never exercise
(an instance of "the single production call site is the weakest point of an
injected dependency").

**How to apply:**
- Pass `os.Stdout` explicitly for every report and data output; reserve
  `fs.Output()` for usage and flag errors.
- Add one `<cmd> 2>/dev/null` run to the pre-release simulation — a mechanical
  check that nothing meant for stdout disappears.
- Under util-series' "stdout is data, UI/diagnostics are stderr" rule, a text
  rendering of the same content as `--json` is data.

### A child process is told whom it serves by parameters on its launch line, never by guessing

**Symptom:** an MCP server spawned by a runtime (Claude Code / gem-agent)
had to know which session it served. The first cut guessed the runtime
from the child's environment (`CLAUDE_CODE_SESSION_ID` /
`GEMAGENT_WORK_DIR`); a gem-agent started from a Claude Code shell then
handed its MCP child the **outer Claude Code session** — environment is
inherited across nested starts. A second cut picking the "nearest"
runtime by parent pid only patched that case and invented the next.

**How to apply:**
- Give the child's CLI **explicit parameters** — `--agent <name>`,
  `--session <id>`, `--project <dir>` — passed on the registration line
  (mcp.json / hook config). The child's code knows no runtime's variable
  names.
- **Leave expansion to the runtime**: a runtime that expands `${VAR}` in
  its config from its own process environment (gem-agent's mcp.json,
  after its own `Setenv`) gets a literal. A runtime that **sets a value
  for children only** (Claude Code sets `CLAUDE_CODE_SESSION_ID` for
  children but expands `.mcp.json`'s `${…}` from its own environment —
  measured: `${CLAUDE_PROJECT_DIR}` stayed literal and the nested case
  yielded the outer session id) gets the **variable name**, e.g.
  `--session-env <VAR>`, and the child reads its own environment.
- A value that resolves to nothing (empty, an unexpanded `${…}` or
  `$VAR`, whitespace) is never adopted: degrade to "unknown" and say so
  in results and usage. Adopting an unexpanded `$VAR` makes every
  instance started from that line one shared identity.
- The runtime exports **the same set of facts** a child needs (session
  id, project directory, work directory) and guarantees the identifier
  is unique (gem-agent ADR-0071: timestamp ids were unique only per
  project, so they became UUIDs).
- Measure with a probe server that only dumps its argv and environment,
  run under `-p`. Whether a value expands, and from whose environment,
  cannot be known any other way.

### SQLite FTS5's default tokenizer makes Japanese one token — use trigram, and scan for terms under three characters

**Symptom:** an FTS5 table built with the default tokenizer (unicode61)
splits only on whitespace and punctuation. "gem-agent からの返信テスト"
made "からの返信テスト" one token, and `MATCH '返信テスト'` returned
nothing. Only words that happened to sit between brackets matched, so
English-only tests never noticed; the first Japanese record in real
use did.

**How to apply:**
- Build an FTS5 table that must search Japanese with
  `tokenize='trigram'` (SQLite 3.34+, available in modernc.org/sqlite).
  It matches any substring of three or more characters in any script
  and folds case by simple Unicode mapping. Migrate an existing table
  with DROP → CREATE → `INSERT INTO t(t) VALUES('rebuild')` from the
  content table (recreate the triggers of an external-content table).
- Trigram **never** answers a term under three characters. When a plain
  query carries one, switch to a scan in the application. Fold case for
  the scan **in Go** (`strings.ToLower`): SQLite's `LIKE` and `lower()`
  fold ASCII only without ICU, so the answer would otherwise depend on
  which path the query took.
- Substrings match inside longer words ("warm" in "swarm"). Show the
  body in the result and let the reader verify.
- Write the tests with Japanese bodies. English-only tests pass the
  default tokenizer's defect through.

## Do not gate the next send on an acknowledgement the peer is not obliged to send

**What happened:** A remote-desktop client sent the guest a monitors
configuration and then refused to send another until the first was acknowledged.
The transport it ran over consumes that message in the hypervisor and never
replies, by design. The first request therefore latched the sender for the life
of the connection: every later resize was queued and dropped, while the session
kept reporting that resizing was available, so the failure was silent for the
operator. Reconnecting cleared it, which is why manual testing never caught it
(spice-client, 2026-09-18).

**How to apply:** A send window of one, held open by a reply, is only safe when
the protocol *requires* the reply. When it is optional, or when one transport of
several omits it, the window needs a second way to close: a timeout, a
supersede-in-place rule for a request whose newer version makes the older
irrelevant, or no window at all. The shape to look for in review is a state
machine whose "in flight" field has exactly one clearing path, and a capability
flag that keeps advertising the feature while that field is stuck.

The fix that kept everything worth keeping was a deadline on the in-flight entry
rather than removing the window: coalescing still protects the peer during a
drag, and liveness is restored for the case that matters, a newer request
waiting behind an older one. Two details make it safe. An expired window with
nothing queued must send nothing, or liveness becomes a resend loop of state the
peer already has. And a reply arriving after the expiry acknowledges whatever is
in flight then, so if those events are surfaced, say that the newest wins rather
than pretending the correlation survived.
