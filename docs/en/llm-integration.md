# LLM Integration

Lessons from integrating Vertex AI Gemini, local LLMs, and agent workflows into
tools. Each entry follows **symptom → why → how to apply**. Core principle:
**LLM output is a system boundary and requires defensive code.**

---

## SDK & API

### Vertex AI: use google-genai (Python) / google.golang.org/genai (Go)

**Symptom:** The old SDKs (Python `vertexai`, Go
`cloud.google.com/go/vertexai/genai`) were deprecated and then removed in
2026-06. The old Go SDK printed runtime WARNINGs to stderr during deprecation,
breaking stdout-JSON + stderr-control pipelines.

**How to apply:**
- Python: `google-genai>=1.0,<2.0`,
  `genai.Client(vertexai=True, project=..., location=...)`; structured output via
  a Pydantic model in `config.response_schema`.
- Go: `google.golang.org/genai` + `genai.BackendVertexAI`.
- **`client.files.upload()` is Developer-API-only** — it raises `ValueError` on
  a Vertex client. Pass media inline with
  `types.Part.from_bytes(data=..., mime_type=...)` (nothing persists server-side,
  no cleanup needed).

### Gemini 3 requires echoing back thoughtSignature

**Symptom:** Switching the backend to Gemini 3 broke the tool-call loop on round
2 with `400 INVALID_ARGUMENT` ("function call ... is missing a
thought_signature"). Response parsing extracted only `FunctionCall.Name/Args` and
dropped `Part.ThoughtSignature` (Gemini 2.5 never emitted signatures, so the same
code used to work).

**How to apply:**
- Gemini 3+ attaches an opaque continuation token (`thoughtSignature`) to each
  response Part and requires **echoing it back on the same Part** when replaying
  history.
- Signatures can ride on all three Part shapes — thought / text /
  function-call — capture and replay all of them. SDK Part constructors don't set
  the signature; assign it separately.
- `IncludeThoughts = false` still emits signatures. Even if you filter thought
  text, keep and replay the signatures.
- Persist signatures through session export/import too. Old sessions without
  signatures fail with 400 on their first multi-call round under Gemini 3 —
  unmigratable; advise starting a new session.

### The Gemini API throws 429s under heavy sequential load

**Symptom:** A large analysis making dozens of sequential LLM calls hit constant
429s; cloud inference-speed advantages were cancelled by rate-limit waits,
yielding local-LLM-level throughput.

**How to apply:** Don't casually propose the Gemini API as the cloud alternative
for high-volume sequential workloads. Decide by frequency × data volume ×
sensitivity (high frequency favors local LLMs; enterprise-contracted managed
services tend to have stabler limits).

### Vertex AI tool config: unify the schema, keep paths per-tool

**Symptom:** Tools varied wildly in configuration style (env-only / .env /
config.toml), creating operational load. During unification, a shared config path
was deliberately **not** created.

**How to apply:**
- Unify the **TOML schema (`[gcp]` project/location + `[model]`), loader
  pattern, and env precedence** (`<TOOL>_<FIELD>` > `GOOGLE_CLOUD_<FIELD>` >
  config file > default).
- Keep **paths per-tool** (`~/.config/<tool-name>/config.toml`) — preserving each
  tool's ability to point at an independent GCP project (staging vs production).

## Defensive output handling

### Validate semantic consistency, not just schema

**Symptom:** Output that is schema-valid but semantically contradictory (critical
findings present yet verdict approved) passes Pydantic validation. Prompt
injection can induce exactly this shape.

**How to apply:**
- Add rule-based post-validation (e.g. `critical > 0 && approved` → force
  rejected + audit log).
- Delegate to the LLM only what tolerates hallucination (reasoning, synthesis);
  **do aggregation and consistency checks in code**.

### Never trust citation excerpts — recover them from the source

**Symptom:** In real-data verification, every LLM-generated citation excerpt had
omitted fields or altered values. The model produces "plausible" partial quotes
with arbitrary field selection, losing context analysts need.

**How to apply:**
1. Check the excerpt's values against the original (relevance check).
2. Relevant → **replace with the full original** (no partial quotes).
3. Not relevant → hallucination warning + show the original for human judgment.
4. Empty citation → recover from in-text references (Record #N).

### LLMs repeat the same observation in different words — 3-tier dedup

**Symptom:** Accumulating stores (findings, memory extraction) fill with
paraphrased duplicates; exact-match dedup misses them.

**How to apply:** Dedup at insert time in three tiers:
1. Exact match
2. Normalized match (lowercase + whitespace collapse + non-letter/digit →
   space)
3. **Word-set Jaccard ≥ 0.5** (ASCII letter/digit runs ≥3 as tokens, CJK runs as
   3-char n-grams — the hybrid matters; either alone misjudges mixed-language
   text)

0.5 is the empirically good threshold: real LLM duplicates score 0.5–0.65,
genuinely distinct observations fall under 0.4 (0.7 misses real dupes). Tell the
caller (including the LLM) about dedup hits ("already recorded") to prevent
cosmetic-rewording retries.

### Single-list-field wrappers drop intermittently (bare-array drift)

**Symptom:** Requesting `{"tactics": [...]}`, Gemini intermittently returns the
bare top-level array `[{...}]`. The structural mismatch survives jsonfix and
fails the whole stage at unmarshal. Never reproduces against fakes — the classic
drift that only real E2E surfaces.

**How to apply:** Implement `UnmarshalJSON` on single-list-field types: if the
first non-space byte is `[`, decode straight into the slice; otherwise decode as
the wrapper via an alias type. Types with 2+ fields don't degrade to bare arrays.

### Quirks of long audio / long JSON responses, and defenses

**Symptom (observed in Gemini audio transcription):**
1. Despite "absolute seconds" instructions, a few percent of segments came back
   with `end < start` (duration notation).
2. Long responses truncated/corrupted at the end (finish_reason still STOP).
3. Timestamps drifting past the actual audio length (model switching doesn't
   reliably fix it).
4. **Mid-stream corruption** too, not just truncation (missing commas, unescaped
   quotes inside one segment). Happens even with
   `response_mime_type=application/json` + `response_schema`.
5. JSON repair tools can fix **syntax while destroying meaning** (empty keys,
   missing required fields). Repair success ≠ correct recovery.

**How to apply:**
- Normalize timestamps (`end < start` detect → repair + warn). For
  sample-accurate uses, don't trust LLM timestamps — get real durations and use
  forced alignment or pre-chunking with re-offsetting.
- **Validate list elements one at a time and salvage (never all-or-nothing)** —
  one corrupt element must not destroy minutes of paid results. Drop bad items
  with warnings and return partial results; hard-error only on zero valid items.
  Offer `--strict` to opt out.
- Keep response_schema as a prevention layer, but write code assuming no JSON
  guarantee.

### Speaker names can be inferred from context (but forbid fabrication)

**Symptom:** Without hints, speaker names can be extracted correctly from
in-audio context.

**How to apply:** Prompt with three explicit cue types: self-introduction /
direct address / third-party mention followed by that person speaking. Also state
"fall back to Speaker A/B/C when unidentifiable" and "never fabricate names".

## Token & budget management

### Word-based token estimation undercounts JSON 4–5×

**Symptom:** JSON symbols (`{` `}` `"` `:` `,`) don't count as words; the gross
underestimate packed far too many records into one window and response quality
collapsed.

**How to apply:** Compute both word-based (CJK×2 + ASCII words×1.3 + punct) and
char-based (chars/4) estimates and **take the max**. Add a
max-records-per-window quality guard (even within token budget, too many records
degrade quality).

### Budgets need enforcement at prompt-build time, not just calculation

**Symptom:** A token budget was computed but the prompt builder passed the full
data anyway. Citation excerpts embedding whole original records ballooned to 4×
budget, processing time grew stepwise, and the run hit timeouts.

**How to apply:** Wherever accumulated data enters prompts, implement budget
calculation and budget application (truncation) as a pair. Structures containing
raw-data references must be controlled by tokens, not item counts.

## Pipeline design

### Upstream LLMs translate prompts to English — detect the user language independently

**Symptom:** User asks in Japanese → the assistant LLM translates the tool-arg
prompt into English → the downstream LLM outputs English → the user receives
English results. "Keep the language" instructions inside the prompt get
overridden by the upstream model.

**How to apply:** Detect the user's language heuristically at the agent layer
(CJK character ratio ≥ 30% of letter/digit runes in the latest user message —
hiragana/katakana/CJK Unified/Ext A) and pass it downstream as an explicit
LanguageHint. Downstream prompts say "input may have been translated upstream;
ignore that and write in <hint>".

### Count "recent N records" on non-tool records

**Symptom:** Taking the flat tail-N of conversation history let tool-call bursts
fill the whole window with tool records; the extraction LLM found nothing and
returned NONE repeatedly (surfacing as "memories never get added").

**How to apply:**
- Walk backwards collecting user/assistant records until the target count
  (exclude tool records from the prompt). Always cap the walk (prompt-bloat
  protection).
- Build observability in from the start: log the extractor's raw response
  (truncated) and each parse-failure / allowlist-drop / dedup-drop case.
  Otherwise "why is nothing extracted" is a black box.

### Thinking modes can degrade structured analysis

**Symptom:** In a phishing-detection eval, enabling thinking/reasoning mode
consistently lowered accuracy across all tested models (one dropped 100% → 80%).
The reasoning over-thinks clean indicators (SPF pass etc.), talks itself into
"safe", and misses content-level signals.

**How to apply:** For classification/analysis tasks with structured prompts and
pre-computed indicators, recommend thinking OFF. The guidance suffices without
intermediate reasoning.

### Don't reuse cloud prompts for local LLMs

**Symptom:** Prompts optimized for cloud LLMs dropped sharply in accuracy on ≤26B
local models (measured 70% → 100% after local-specific optimization).

**How to apply:**
1. **Shorter** — long instruction lists confuse small models.
2. **Affirmative** — "X is safe" beats "X is NOT suspicious".
3. **Interpret indicators explicitly** — spell out conclusions like
   "authentication ALL PASS + sender clean → likely safe".
4. State domain-specific decisive rules explicitly (e.g. suspicious link +
   password request = phishing regardless of authentication).

## Agents & subagents

### Subagent output contracts must state what NOT to output

**Symptom:** Instructed only with a schema and "what to report", multiple
subagents helpfully mixed **out-of-scope observations** into the regular output
format with pseudo-references. Format validation cannot stop "out of scope but
formally valid" output.

**How to apply:** Add one or two lines of negative-space boundary to the
contract. Also (1) give out-of-scope findings a **correct destination** in the
contract (prohibition alone silently loses information), and (2) have the
aggregator expect and handle stray out-of-scope items.

### Don't assign write tasks to background agents

**Symptom:** Background-run agents couldn't obtain interactive approval for
write permissions and stalled without producing artifacts.

**How to apply:** Run tasks involving file creation/editing in the foreground;
restrict background agents to read-only research.

## Documentation & licensing

### Never claim "fully offline" for tools running in an LLM session

**Symptom:** A Skill's README claimed "fully offline / completes locally" and was
called out as an overstatement. Content entering the session (even preprocessed)
is sent to the model backend. For security tools handling sensitive data, such
overclaims mislead users' data-handling decisions.

**How to apply:**
- Phrase claims by **enumerating destinations**: "the skill itself makes zero web
  accesses and zero connections to the target; content reaches only the
  session's model backend and local output".
- "Offline" is only valid when the process truly runs with no external
  connections (local LLMs etc.). Distinguish the tool's own traffic from the
  session's traffic.
- Review trigger: any "offline / local-only / no external transmission" claim
  gets fact-checked including model-backend transmission.

### Derive model licenses from the upstream weight distributor

**Symptom:** Judging by a repackage repo's missing LICENSE led to a
"license-unclear" misclassification; the upstream repo and HF model card declared
Apache-2.0. Over-conservative classification has real cost — it chills
legitimate commercial use.

**How to apply:** (1) Treat the HF model card's `license:` metadata as the
primary source; (2) check upstream even when pulling via a repackage; (3) code
license and weight license can differ (the weight side governs). Use
"needs review" only for genuinely undeclared cases.
