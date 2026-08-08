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
