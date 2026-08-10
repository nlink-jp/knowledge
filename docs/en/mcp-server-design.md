# MCP Server Design

Lessons from implementing MCP (Model Context Protocol) stdio servers in Go, and
from designing tools called by LLM agents in general. Each entry follows
**symptom → why → how to apply**.

---

## Protocol & transport

### The MCP protocol has no tool-call cancellation

**Symptom:** User report: "aborting the chat doesn't work while an MCP call is in
flight." Investigated how to interrupt an in-flight tool call.

**Why:** The MCP 2024-11-05 spec has no client→server cancel notification. Once
dispatched, a tool call runs to upstream completion. Ad-hoc protocol extensions
only help cooperating servers and don't unblock a blocked stdio read.

**How to apply:**
- The only client-side way to "stop waiting" is **killing the child process and
  closing stdin to unblock the Scan** (kill-and-respawn). A context-aware wrapper
  is `goroutine + select(ctx.Done, result)` with Stop()-kill on cancel.
- After killing, the client owns re-spawning **that server only**, asynchronously
  (restarting all servers drags in unrelated ones).
- Upstream side effects (external HTTP / DB) don't stop on kill — the server
  completes and the client discards the result. Protocol-level unavoidable;
  document it to set expectations.
- If protocol-level cancel arrives later, the kill approach becomes unnecessary —
  keep the wrapper easy to swap.

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
- Offer the untrimmed result as a file when a workspace is available; trimming
  should cost a file read, not the data.
- Put the cap in the **face that has the budget**. A CLI's `--json` feeding a
  pipeline should stay complete; only the tool response needs bounding.

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
