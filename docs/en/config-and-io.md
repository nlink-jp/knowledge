# Config, Identifiers & CLI IO

Config-file loading, identifier canonicalization, OAuth, and terminal IO. Each
entry follows **symptom → why → how to apply**.

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
