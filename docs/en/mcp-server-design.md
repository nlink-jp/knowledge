# MCP Server Design

Lessons from implementing MCP (Model Context Protocol) stdio servers in Go, and
from designing tools called by LLM agents in general. Each entry follows
**symptom → why → how to apply**.

---

## Protocol & transport

### MCP's cancellation notification may be ignored by the receiver — killing the child process is still the only reliable stop

**Symptom:** User report: "aborting the chat doesn't work while an MCP call is in
flight." Investigated how to interrupt an in-flight tool call. The original
record asserted that "the MCP 2024-11-05 spec has no client→server cancel
notification" — **that was wrong** (corrected 2026-09). The spec has defined
`notifications/cancelled` (`requestId`, `reason`) since its first published
version, 2024-11-05.

**Why:** The error came from not consulting the primary source (the schema or the
relevant chapter) and inferring the protocol's absence from a code comment in the
in-house stdio client ("the only way to unblock a blocked Scan is Stop()"). An
implementation's circumstances are not evidence about the specification. The
conclusion nevertheless stands: per the spec, a receiver **MAY ignore** the
notification when the request "cannot be cancelled", and the in-house stdio
skeleton, like most third-party servers, discards it. Design as if a dispatched
tool call runs to upstream completion.

**How to apply:**
- The reliable client-side way to "stop waiting" is **killing the child process
  and closing stdin to unblock the Scan** (kill-and-respawn). Sending
  `notifications/cancelled` before the kill is fine, but never expect the send
  alone to stop anything. A context-aware wrapper is `goroutine +
  select(ctx.Done, result)` with Stop()-kill on cancel.
- After killing, the client owns re-spawning **that server only**, asynchronously
  (restarting all servers drags in unrelated ones).
- Upstream side effects (external HTTP / DB / billing) don't stop on kill — the
  server completes and the client discards the result. For long-running,
  metered tools keep the call's own caps small and document the expectation.
- If a server ignores the notification, say so in its comments as "received and
  ignored", not "absent from the spec".

### Proxies must never leave a client request unanswered

**Symptom:** A proxy logged upstream-forwarding failures (expired OAuth token)
without responding to the client, which hung for 15 seconds and surfaced an
inexplicable `read response: EOF`.

**Why:** Other failure paths (tool not found, etc.) returned JSON-RPC errors, but
the upstream-forwarding-failure path was missing a response. Asymmetric failure
paths always leak.

**How to apply:**
- Invariant: "any request whose routing returned an error gets a JSON-RPC error
  response." At the central dispatch point, if `msg.IsRequest()` (has an id),
  write an error response. Notifications (no id) can only be logged.
- Put an actionable reason in `err.Error()`
  (`access token expired ... run --login again`) so clients can surface it as-is.
- Watch for double responses: handlers either write a response or return an
  error, never both. The central catch only handles errors that propagated
  unanswered.
- If the timeout path already converts to error responses, align the
  send/connect failure paths the same way (leave no asymmetry).

### Surface child-process exit status, not pipe symptoms

**Symptom:** A child process (MCP server) was being SIGKILLed by macOS, but the
error surfaced only as `initialize: read response: EOF`, making an environment
problem look like a code bug and costing significant diagnosis time.

**How to apply:**
- On failure, include `cmd.Wait()`'s result (`signal: killed` /
  `exit status N`) in the error. For SIGKILL, add a remediation hint ("if under a
  quarantine/synced path, macOS may be killing it") for one-shot triage.
- **os/exec pitfall**: `cmd.Wait()` must not race the `StdoutPipe`/`StderrPipe`
  reads. Call Wait **once, from the goroutine that owns the pipe reads, after
  the read loop ends at EOF**. A separate Wait-watcher goroutine hits exactly
  this race.
- Concrete shape: the stderr-drain goroutine, after Scan ends (= process exit),
  runs `exitErr = cmd.Wait(); close(exited)`. Failure paths wait briefly on
  `exited` and wrap exitErr **only for closed-pipe-class errors**; failures while
  the process is alive (upstream JSON-RPC errors) must not wait. Stop() only
  kills (no Wait) to avoid double-Wait.

### stdio servers embedding native libraries must isolate stdout at the fd level

**Symptom:** A statically linked native library (C/CGO) printf'd a progress bar
**to fd 1** during model load, corrupting the JSON-RPC stream. On a TTY the
`\r`-updating line is invisible; only in a pipe (= MCP) do the raw bytes remain.
Unit tests with a stub build never reproduced it; a real-engine E2E finally did.

**Core lesson:** "not visible on the console ≠ not written to stdout." Always
verify with `1>file 2>file` stream separation.

**How to apply (two layers):**
- **① Plug the source**: for libraries that "draw to stdout when no callback is
  registered", always register a callback — even a no-op.
- **② fd-level isolation (defense in depth)**: at startup, dup the real stdout
  into a private handle and repoint fd 1 at stderr; the transport writes only to
  the private handle.

```go
saved, _ := syscall.Dup(int(os.Stdout.Fd()))
syscall.Dup2(int(os.Stderr.Fd()), int(os.Stdout.Fd()))
mcpOut := os.NewFile(uintptr(saved), "mcp-stdout") // transport writes here
```

- Log to `os.Stderr` (after repointing, even `fmt.Println` lands harmlessly on
  stderr).
- Child-process architectures (driving external engines as subprocesses) don't
  have this problem — it's specific to in-process embedding.
- Verification requires a dummy-stdio-client E2E against the real engine (see
  testing.md). Contamination shows as blank/non-JSON lines in the response
  stream.

### RFC 9728 metadata coming back does not mean DCR is available

**Symptom:** An upstream MCP server answered 401 with
`WWW-Authenticate: ... resource_metadata=...`, the protected-resource metadata
(RFC 9728) fetched correctly, so a bridge was configured to "discover the
endpoints and register a client automatically" — and the login never completed.

**Why:** Discovery is two independent stages. The first (RFC 9728 → identify the
authorization server → fetch its RFC 8414 metadata) can succeed while the
second, dynamic client registration (RFC 7591), cannot happen at all: it only
works when that metadata carries a `registration_endpoint`. Large providers
routinely omit it, leaving no option but registering an OAuth client by hand.
The trap is that metadata coming back *looks* like proof the automatic route
will work.

**How to apply:**
- Before trying automatic discovery, **fetch the metadata and check whether
  `registration_endpoint` is present**. If it is absent, only the pre-registered
  route (client_id / client_secret / fixed redirect URI) exists. One curl
  settles it, faster than inferring it from a failed login.
- When the authorization server's issuer has a path, RFC 8414 inserts the
  well-known segment *before* that path (issuer `https://host/login/oauth` →
  `https://host/.well-known/oauth-authorization-server/login/oauth`). Appending
  it to the issuer, OIDC-style, returns 404 — try both shapes.
- A provider without `registration_endpoint` may still accept a token an
  existing CLI already holds, or a personal access token. A route that obtains
  the token from an external command avoids registering an OAuth app entirely.
- An implementation that reports a discovery failure as merely "not logged in"
  hides the cause. Say **which stage failed**.

### A synchronous write to a child's stdin is unbounded — outside the deadline, one wedged peer stalls everything

**Symptom:** A stdio client wrote the request, then started a deadline and waited for the
response. When the peer stops reading its stdin the pipe fills and `Write` never returns. The
deadline is created **after** the write, so nothing supervises that wait — and the write lock is
held across it, so every other call to that server parks behind it. The timeout branch that
kills the child is unreachable from all of them, so it never recovers.

**Why:** There was a test for "the server receives a request and does not answer". That never
exercises the write at all: it begins after the request has landed.

**How to apply:** Create the deadline **before** the write, covering both halves. On expiry kill
the child — closing the read end returns the parked `Write` with EPIPE and releases the lock.
Put the fix in the **send function**, not the caller: every sender has the same hole, and a
reply written from the read loop has no call to inherit a deadline from. Test with a server that
stops reading stdin, and with a second caller waiting on the lock.

## Error & schema design

### Return tool errors as structured {code, message, details} JSON

**Symptom:** Plain-string errors force LLM clients into string matching, making
prompts and implementations fragile.

**How to apply:**
- Make the text content under `isError: true` structured JSON:
  `code` (stable slug: `"path_not_allowed"` …) / `message` (human-readable) /
  `details` (machine-readable context:
  `{"requested": "bash", "supported": ["python"]}`).
- Go shape: a toolerr package with an Error type + Code constants, `Is(target)`
  comparing Codes to keep `errors.Is` compatibility, package-level sentinels;
  the server does `errors.As` → JSON marshal → text content, with an
  `err.Error()` fallback for unstructured errors.
- Adding new codes is compatible; **renaming existing codes is a breaking
  change**.

### Clients pre-validate inputSchema.enum — keep server-side validation anyway

**Symptom:** Enum-violating calls never reached the server handler; the client
(Claude Desktop, measured) rejected them as `invalid_enum_value`.

**How to apply:**
- Declare enumerable arguments in `inputSchema.enum` (good UX: rejection without
  a round trip).
- **Also validate server-side** — defense in depth for clients that don't
  validate schemas and for raw JSON-RPC test harnesses.
- Testing the server-side enum path requires bypassing via a dummy JSON-RPC
  harness (real clients never reach it).

### Rich content (images) via a sentinel type + an explicit return tool

**Symptom:** The standard path (marshal handler result → single text block) can't
return images. And "auto-scan the work dir after execution and return new
images" is trap-laden: unintended files get returned, mtime/filename heuristics
are fragile, and response size is unpredictable.

**How to apply:**
- **RawResult sentinel pattern**: introduce a dedicated type
  (`Content []ContentBlock, IsError bool`); the dispatcher checks
  `if raw, ok := out.(RawResult); ok` and passes multiple blocks through.
  Existing tools stay unchanged (backward compatible).
- **Let the LLM name what to return** with a dedicated tool like
  `attach_files(workspace_id, paths)` — controllable, auditable, easy to enforce
  size limits and path-traversal defense (`filepath.Clean` + prefix check).
- Monkey-patching library functions to hook output is too opaque to debug —
  reject that option.

## Designing tools called by LLMs

### Never pass large data through tool arguments (use a work directory)

**Symptom:** Passing document bodies via a `content` parameter broke down on
large files. LLM function-call arguments have practical size limits (hundreds of
KB; smaller for local LLMs). This is **the primary failure mode of tool design**.

**How to apply:**
- Make `filename` + a shared work directory the canonical pattern: an upstream
  tool stages the file; downstream tools reference it by basename.
- **Beware ambiguous parameter names**: with `content`, LLMs confuse "the user's
  message" with "the document body" (observed: the user's request text itself was
  passed in for summarization). Prefer clearly typed names like `filename` and
  state "NOT the user's request text" in the description.
- **Interpret XOR constraints leniently**: treat empty string `""` as "not
  provided" — LLMs often express "unused" as an explicit empty value.
- **Path-traversal defense**: accept basenames only (reject `/` and `..`). LLMs
  generate absolute paths without malice.
- Keep all LLM guidance in the tool description field (wiring specific tools
  into the host's built-in prompt raises coupling). Describe when to choose the
  tool, workflow chains, parameter intent, and exclusivity constraints.
- If a wrapped CLI logs progress to stderr, suppress it (`--quiet`).

### A `limit` that caps one list does not bound the response

**Symptom:** A lookup tool's `limit` argument capped the list of records it
returned, and a call with `limit: 3` still produced a **162 KB result the MCP
client refused outright** — the tool was unusable for exactly the inputs worth
looking at. The record list was 1.5 KB, as asked. The other fields were not
governed by `limit` at all: an aggregate of 1,705 tags (63 KB) and 232
references (32 KB), both derived from the *whole* result set rather than the
part being shown.

**Why:** `limit` naturally attaches to the thing a caller thinks of as "the
results". Aggregates, cross-references and provenance lists are computed over
everything and grow with the popularity of the input, so they are unbounded by
construction. Stub-based tests never see it, because a fixture returns a small
aggregate; the failure needs real data with a long tail.

**How to apply:**
- Budget the **whole response**, not the list the caller named. Ask of every
  field: what makes this small?
- Trim the ranked tail, not an arbitrary slice. An aggregate sorted by
  corroboration loses its least-supported entries first, which is the part a
  reader would discard anyway.
- **Account for every value dropped** — a per-category omitted count plus a
  plain-language note. A silently shortened answer is worse than a refused one,
  because nothing signals that a view is partial.
- Emit the accounting fields **only when something was trimmed**, so their
  presence is itself the signal.
- Make the trim **escapable in place**: a knob that raises the cut (`context_top`,
  `references_top`, `-1` for none) keeps the tail reachable in the same result.
  Do not reach for a file here — see the next entry.
- Put the cap in the **face that has the budget**. A CLI's `--json` feeding a
  pipeline should stay complete; only the tool response needs bounding.

### A server cannot know the model's context window

**Symptom:** Seven file-mediated servers in one fleet had independently grown a
`workspace_root` argument: past a threshold each wrote its result to a
caller-supplied directory and returned the path. The client calling them had no
size cap of its own on tool results, so every server was implementing the
missing guard — seven times, in the seven processes least able to do it.

**Why:** "Too big" is a property of the caller's context window and of the model
behind it, and a server can observe neither. Its threshold is therefore a guess
that is wrong for every client but the one it was tuned against. The escape
hatch costs more than the guess: it makes the server depend on the client owning
a filesystem it can *name*. A client speaking MCP over a transport with no
shared disk cannot read its own result, and a sandboxed client cannot open a
path outside its roots.

**How to apply:**
- **Bounding a response is the caller's job; spilling an oversized one is the
  client's.** Give the caller a knob that bounds the response (`per_page`,
  `limit` + `offset`, a `*_top` cap), make every value reachable through it, and
  then return what was asked for whole and inline.
- Preserve **reachability**, not the file. A file that only ever held what
  `limit` had already fetched reached nothing further; pagination that walks to
  the end replaces it exactly. Check this before removing a file path — if the
  file did reach further, add the paging first.
- Cap on the client side once, where tool results enter the conversation, and
  **save rather than truncate**: hand the model a preview plus a path it can
  open. A built-in tool cap that truncates while MCP results pass through
  unbounded is the worst of both.
- Binary is the one case where a file is intrinsic — audio, video, a rendered
  image. Even there, prefer MCP content blocks when the payload fits the
  transport: a path is useless to a client that cannot read the server's disk.
- Dropping file mediation removes a dependency, not a capability. Say so in the
  CHANGELOG with the migration (a smaller page plus a loop), because the
  argument disappearing from the schema is a breaking change.

### Don't expose internal IDs in tool results

**Symptom:** Including `[Stored as object ID: %s]` in a tool result made the LLM
write redundant links into chat replies (the content was already displayed via a
dedicated UI path).

**How to apply:**
- Include only information the LLM needs for **subsequent tool calls** (IDs used
  by a follow-up tool stay).
- Exclude IDs of content already shown to the user in dedicated UI.
- Convey "display was handled" with minimal status
  ("SUCCESS: ... displayed to the user. Reply briefly.").

### A tool list is context paid for every session — let clients pick a subset at connect time

**Symptom:** An MCP server returning 44 tools answered `tools/list` with roughly
121 KB. Barely a dozen were ever used; the rest were context read on every
session and nothing else.

**Why:** `tools/list` is read in full when a session opens, and the model carries
that whole text as context. At a few hundred bytes of description each, a few
dozen tools become a major part of the prompt. The weight is not proportional to
the count either — it concentrates in the **few tools with long descriptions**.
Measured: narrowing by group (category) from 44 to 38 tools moved 121 KB only to
106 KB, while naming 12 individual tools dropped it to 32 KB. Group-level
selection barely helps.

**How to apply:**
- A server with more than about 20 tools should offer **a way to select a subset
  at connect time** — a request header, a URL path, or a launch flag. The useful
  axes are allowlist, denylist, and read-only.
- Group (category) selection alone is not enough. Always offer **selection by
  individual tool name**.
- Do the narrowing **upstream, at the server**. Discarding tools in a relay or
  client still pays for the transfer, and in a transparent relay it also breaks
  the invariant that messages are never interpreted.
- Decide and document how unknown names behave. An allowlist that fails to start
  on an unknown name while a group selection ignores one silently is an
  asymmetry users cannot discover.
- Name lists stop matching silently when the upstream renames a tool. Document
  them as something to re-check, not to set and forget.

### Enumerate the capability surface before building a control layer

**Symptom:** A requirement — "make the worst operation (deleting a repository)
unreachable" — led to a plan for hiding tools in a relay. Enumerating all 89
upstream tools first showed that **no repository-deletion tool existed**. There
was no branch, tag or release deletion and no force push either; the only
destructive primitive was file deletion.

**Why:** "What the API permits" and "what the MCP server exposes as tools" are
different sets, and the second is usually far narrower. Building a control layer
against a threat imagined from the first leaves a feature with nothing to
protect — and an unused feature is maintenance cost on its own.

**How to apply:**
- Before designing controls, fetch the whole `tools/list` and scan **both names
  and descriptions** for destructive words (delete / remove / destroy / archive /
  transfer). Deletions hide behind names that do not say so (a method argument
  of some `*_write` tool), so the descriptions matter.
- When a permanent limit is required, impose it through the **credential's
  authority, not a denylist of tool names**. A name list is bypassed the moment
  the upstream adds or renames a tool; a credential without the permission
  cannot execute one even after it ships.
- Record the enumeration with its date. Tool inventories change on the
  upstream's schedule, so write "absent as of <date>".
- Before adding a feature to a relay, check whether the upstream already
  provides it. If it does, not implementing is the right answer.

### Discarding a type assertion on a required argument turns absence into a destructive default

**Symptom:** A string-replacement tool's batch form read `m["new_string"].(string)` and threw
away the ok. A missing key, `null`, or a number all produced `""`, which the later checks did
not reject — so `{"old_string": "something important"}` executed as a **deletion** of that text.
The single-pair form had the required check; the two forms disagreed.

**Why:** The `required` list in the schema handed to the model does not protect the execution
side. And a zero value is not necessarily a safe default: empty string means delete, false means
"require uniqueness", 0 can mean "unbounded". Where one tool has two parse paths, only one of
them tends to carry the check.

**How to apply:** Check presence and type together for required arguments (in Go, both
`v, ok := m[k]` and `s, ok := v.(string)`). Where a zero value is destructive, distinguish
missing, wrong-typed, and deliberately empty. Then reduce it to **one parse path** — treat the
single form as the batch of one and run it through the same function. Keeping two
implementations in agreement by inspection fails.

## Contracts with upstream APIs

### Retry safety comes from a resource you named, not from a token whose meaning you assumed

**Symptom:** An MCP server for BigQuery was designed on the assumption that
`jobs.query` with a `requestId` makes one retry idempotent. An independent
review re-read the Discovery document: `requestId` deduplicates **mutating
queries only** (read-only queries are "nullipotent" with respect to data —
not with respect to cost). Lose the response after the request has landed,
and a SELECT runs twice and bills twice.

**Why:** Idempotency-token semantics differ per API, and the phrase "reads
may ignore this token" is easy to misread as "re-running a read is free". A
design that delegates safety to an interpretation of the upstream looks
correct until someone re-reads the document.

**How to apply:**
- Create anything you must not run twice as a **resource the client names**
  (for BigQuery, `jobs.insert` with your own `jobId`). The second attempt
  answers `409 duplicate` and you continue with the resource that exists.
  Choose the name before the first attempt and resend the same one.
- Read the full description of any upstream idempotency token (`requestId`
  and friends) and quote the covered operations and guarantee into the
  design record. "Optional for reads" is not "reads are free to repeat".
- An API that calls itself read-only still has side effects — billing,
  audit trails, rate quotas. Decide retry policy by side effects, not by
  the label.

### When the paging response carries no statistics, read the resource itself

**Symptom:** The same server took bytes billed and slot time from the
`jobs.getQueryResults` (results page) response. That response type has no
`totalBytesBilled`, `totalSlotMs` or `statementType`; once the first answer
came back `jobComplete=false`, the values were gone for good. The unit tests
passed because their script completed on the first response.

**Why:** It is tempting to hang the statistics on the single "run, then
page" flow, but the API returns the resource (the job) and its result pages
as different types. The page type is built light, to carry rows, and some
fields exist only on the resource. A scripted test mirrors the designer's
understanding and cannot detect a field that the type never had.

**How to apply:**
- Verify every field you read **mechanically against the type definition**
  (Discovery / OpenAPI); never from "it should be there".
- Read completion statistics once from the resource (`jobs.get`); use the
  results pages for paging only.
- Fake-server tests must include the script "first response incomplete,
  completion only after polling". Scripts that answer as the designer
  expects cannot reveal a missing field.

### An empty string is not "unset" — omit optional fields

**Symptom:** The job's `jobReference.location` was sent straight from the
config (`""` when unset); BigQuery refused it with `Invalid value for
location:  is not a valid value`. The fake server accepted `""`, so every
unit and integration test passed, and the live test against a real project
was the first to fail.

**Why:** Marshalling a Go struct as-is turns an unset string into a field
that is **present** with the value `""`. To the upstream, absent and empty
differ, and inference (here: the location from the datasets) only runs
when the field is absent.

**How to apply:**
- Put `omitempty` on every field the upstream documents as optional, and
  test that the **key is absent** (not that the value is empty).
- A fake server must not be lenient; since making it refuse everything the
  real one refuses is hard, **always ship an opt-in live test** and run it
  once before release. Fake-only tests are a copy of the designer's
  understanding.

## Server implementation structure

### Port a proven skeleton for new Go MCP servers

**Symptom:** Transport / JSON-RPC / protocol routing / error types are generic
parts independent of the service; writing them from scratch each time wastes
effort and varies quality.

**How to apply:** Keep a four-package skeleton and port it at scaffold time:
- `internal/transport/` — stdio: `bufio.Scanner` (large buffer) + `json.Encoder`
  + mutex-serialised writes
- `internal/jsonrpc/` — JSON-RPC 2.0 types + standard code constants
- `internal/mcpserver/` — protocol routing + `RegisterTool` API + structured
  error output
- `internal/toolerr/` — `{Code, Message, Details}` + `errors.Is` by code

Porting means changing the import path, swapping sentinel code constants, and
deleting unused features (e.g. RawResult). Only the tool-handler layer and the
upstream client are new code.

### Dual state (in-memory + disk) must mutate through a single layer

**Symptom:** With a long-lived in-memory session + disk file, a disk-only
mutation (rename) bypassed the in-memory copy. The near-every-action save then
overwrote disk with the stale in-memory value — "rename doesn't stick / reverts
on restart". The same trap was hit twice.

**How to apply:**
- Every per-session mutation goes **through the state-owning layer** (the agent
  layer). UI bindings calling the persistence package directly is forbidden
  (a review red flag).
- Add same-named methods on the state owner that update in-memory under a mutex;
  bindings are thin pass-throughs.
- **Guard-decision trap**: guards reading in-memory values
  (`if Title != "New Session"`) misfire for the same reason. Pin both modes in
  tests: stale-overwrite (Mode A) and wrong-guard (Mode B).

### Serve a huge upstream document as an index first, then page one section

**Symptom:** a single sandbox behaviour summary exceeded 2 MB (a dozen-odd
sections holding thousands of items each) and an ATT&CK tree measured 258 KB —
neither could go into a tool response whole (lookup CLI+MCP project, 2026-09).

**Why:** budgeting the whole response (above) presupposes a list a limit can
bound; an API that answers with *one document* has none, and dropping a ranked
tail does not work either — only the caller knows which section it needs.

**How to apply:**
- Make the first response an **index**: section names and item counts, no
  content. Only scalars (verdicts, flags) ride along.
- Later calls open **one section**, paged by `section` + `offset`/`limit`, so
  every item stays reachable inside the same tool (the reachability rule).
- **Maps are sections too, not just lists.** A live summary carried a 63-key
  map that would have ridden the index and blown the budget; page maps in key
  order as {key, value} entries.
- Fetch and cache the upstream document once; section paging is local — do
  not re-fetch 2 MB per page turn.
- For trees (ATT&CK and kin), default to a **compact identity view** and open
  the raw tree behind `full: true`. Account for what the compact view drops.

### Never cache live account state

**Symptom:** caching the account's own detection-ruleset list looked natural,
but the API's typical caller is someone who just toggled a rule in the console
and asks whether it took (lookup CLI+MCP project, 2026-09).

**Why:** observations of the outside world merely go stale under a TTL; the
account's own configuration is something **the caller just changed**. A cached
"disabled" reports an applied change as ineffective and makes the operator
doubt their own action.

**How to apply:**
- Result caches hold observations of the outside world only. Tools returning
  the account's own configuration or state always answer live.
- For firing conditions like enabled/disabled, state the **consequence**, not
  just the value (enabled: false → "this rule will not fire").
