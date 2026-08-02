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
