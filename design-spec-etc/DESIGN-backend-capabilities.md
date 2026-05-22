# Backend Capability Model

## Core Principle: Routing Is a Performance Contract

The router + StateMachine loop is the nervous system. It needs to be **fast and reliable**. Declaring `:routing` capability means meeting a performance contract — minimum tok/s, maximum latency, availability guarantees. It doesn't have to be local.

A local llama on a Pi? That works. A 16B instruct on a LAN GPU box? Also routes. OpenRouter? Sure — if you accept that your system goes down hard when the internet does. The user decides their risk tolerance. The system just enforces the contract.

Everything else — Processor recipes, Interactive sessions, Engaged reasoning, research threads — can be dispatched to whatever backend fits the cost/capability profile.

## Why Local-Capable Matters

Myelin should be runnable by anyone who cares about privacy and autonomy. Load a couple of models (one tiny + one reasoning) on rented or local GPU and you can run the system. It's naturally better with more capacity connected, but even just that works for many use cases. Users shouldn't need to share their lives with cloud providers overseas to have an AI agent.

The recommended deployment is local routing, but it's not enforced — it's a risk/availability tradeoff the operator makes.

## Capability System

Capabilities aren't just backend labels — they're a **dependency graph** that spans the entire system. Backends declare what they provide. Features declare what they require. Startup validates the graph.

### Primitive Capabilities (what models actually can or can't do)

Capabilities are organized by what they describe about a model.

**Core inference:**

| Capability | What it means |
|---|---|
| `:routing` | Two-pass routing, vibe checks, state transitions |
| `:structured_output` | GBNF / grammar-constrained generation, fixed output formats |
| `:chat` | Conversational interaction with a user |
| `:reasoning` | Multi-step thinking, context-aware decisions |
| `:tool_use` | Function calling, structured tool invocation |

**Processing (mechanical, high-volume):**

| Capability | What it means |
|---|---|
| `:summarization` | Batch summarization, compaction, timeline building |
| `:feature_extraction` | Entity extraction, classification, tagging |
| `:processing` | General Processor recipe execution (catch-all) |

**Modality (input types the model can handle):**

| Capability | What it means |
|---|---|
| `:embedding` | Vector embeddings for semantic similarity |
| `:vision` | Image understanding — OCR, scene description, diagram reading |
| `:audio` | Audio/speech input processing |
| `:code` | Code generation, analysis, execution planning |

**This list is not closed.** Adding a new capability to Myelin is:

1. Add the capability name to the schema (one line)
2. Write a backend config file that declares it
3. Write a manpage so the agent knows what it can do with it
4. Any feature that wants it declares the requirement

No core code changes. No refactoring. The capability graph is extensible by design.

### Composite Capabilities (aliases for common bundles)

| Alias | Expands to | What it means |
|---|---|---|
| `:agent` | `:chat` + `:reasoning` + `:tool_use` | Full agent loop — can converse, think, and act |

Composites are convenience aliases — the system expands them to primitives. A backend can also declare primitives individually. A model that does `:reasoning` + `:tool_use` but isn't great at `:chat` (fine-tuned instruct/expert) is valid and useful.

New composites can be defined as needed:

| Possible future alias | Could expand to |
|---|---|
| `:analyst` | `:reasoning` + `:vision` + `:feature_extraction` |
| `:scribe` | `:chat` + `:summarization` |
| `:multimodal_agent` | `:agent` + `:vision` + `:audio` |

These are illustrative, not prescriptive. Define them when real use cases emerge.

### Two Baseline Assumptions

1. **A reasonably fast model with `:routing` + `:structured_output`** — supports GBNF or grammar-constrained output. This is the nervous system. Small, fast, always-on.
2. **Either 1 frontier model or 1-3 open/free models** that collectively cover `:agent` (`:chat` + `:reasoning` + `:tool_use`). One Sonnet can do it all. Or a capable open instruct model handles `:reasoning` + `:tool_use` while a different fine-tune handles `:chat`. The system doesn't care how you cover the capabilities — just that they're covered.

### Consumer Requirements (features declare these)

**Hard requirements** (system won't start without these):

| Feature | Requires |
|---|---|
| Core (StateMachine + Router) | `:routing` + `:structured_output` |

**Soft requirements** (feature disabled gracefully if unmet):

| Feature | Requires | Without it |
|---|---|---|
| Interactive sessions | `:chat` + `:tool_use` | No user-facing conversation |
| Engaged reasoning | `:reasoning` | Events that need reasoning get queued, not processed |
| Processor recipes | per-recipe (see below) | Individual recipes disabled |
| KnowledgeStore semantic search | `:embedding` | Falls back to keyword search |
| Research threads | `:reasoning` + `:tool_use` | No deep research capability |
| Image understanding | `:vision` | Image events logged but not analyzed |
| Voice interface (future) | `:audio` + `:chat` | Voice events can't be processed |

**Processor recipe requirements (type signatures):**

Every recipe declares a `requires` list — the capabilities a backend must have to run it. The Processor dispatches based on this, never by backend name.

| Recipe | Requires | Notes |
|---|---|---|
| `compress_thread` | `:summarization` | |
| `batch_summarize` | `:summarization` | |
| `draft_session_note` | `:reasoning` | |
| `extract_entities` | `:feature_extraction` | |
| `draft_response` | `:chat` | Drafts for review, not direct output |
| `describe_image` (future) | `:vision` + `:feature_extraction` | |
| (new recipes declare their own) | (extensible) | |

**When a recipe can't be dispatched:**

- Capability exists in pool but all providers degraded → **queue** (transient, will recover)
- Capability not declared by any backend in pool → **error** (permanent, operator must add a backend)
- This distinction is critical: queuing a `:vision` job when no vision backend is configured would silently lose work. Error immediately so the operator knows.

### Dependency Resolution at Startup

The system validates the capability graph on boot:

1. Load all backend configs → collect declared capabilities (with implication expansion)
2. Load enabled features → collect required capabilities
3. **Hard requirements** (`:routing`): if unmet, refuse to start
4. **Soft requirements** (`:embedding`, `:chat`): start with warnings, disable dependent features gracefully
5. Report the resolved configuration to the operator: "Starting with: routing (local-llama), agent+chat (anthropic-sonnet), processing (openrouter-haiku). Missing: embedding — semantic search disabled."

This means you can build Myelin in different shapes:
- **Full-featured:** local routing + Anthropic for chat/agent + OpenRouter for processing + embedding provider
- **Privacy-only:** two local models (small router + large reasoning), no remote calls
- **Headless monitor:** local routing + processing, no chat/agent at all — just watches events and runs Processor recipes
- **Expert system:** routing + agent (instruct-tuned domain model), no chat — processes and acts but doesn't converse

### Beyond Backends

The dependency system extends to everything in the system. Same pattern everywhere: declare what you provide, declare what you need.

- **Interfaces** declare requirements: Telegram needs nothing special. A voice interface needs `:audio` + `:chat`. An image-monitoring interface needs `:vision`.
- **Memory/Knowledge stores** declare what they provide: MemoryStore provides persistence, KnowledgeStore provides retrieval (and requires `:embedding` for semantic mode).
- **Processor recipes** declare per-recipe requirements: `batch_summarize` → `:summarization`, `describe_image` → `:vision`.
- **Manpages** document capabilities for the agent: when `:vision` is available, the agent reads the vision manpage and knows it can ask for image descriptions. No vision? The manpage never loads, the agent never tries.

The trait system is the same everywhere — providers declare capabilities, consumers declare requirements, startup resolves the graph. Adding a new modality to Myelin is a config file + a manpage, not a code change.

### Performance Contracts vs Capability Promises

These are two different things:

**Performance contracts** apply to `:routing` only. A routing backend MUST declare measurable parameters — TTFT, tok/s, latency bounds. The system uses these to set hard timeouts and detect degradation. If a routing call exceeds the declared TTFT, that's a crash, not a slow response.

**Capability promises** apply to everything else. Declaring `:chat` is a promise that the model can hold a conversation. Declaring `:tool_use` is a promise it can do function calling. These aren't measured with stopwatches — they're either true or they're not. The system trusts the declaration and validates through use.

Some capability promises have **structural requirements** — for example, anything declaring `:chat` MUST support either GBNF or JSON Schema structured output, because the async tool call + self-narrated context system depends on structured responses. But that's a format requirement, not a performance metric.

### The `:routing` Performance Contract

`:routing` is the only capability with a hard performance floor. Any backend declaring `:routing` MUST include a `[performance]` block in its config with benchmarked values.

The baseline will be established by benchmarking on a Raspberry Pi server — the weakest hardware we'd actually deploy on. Myelin could ship with a small benchmarking tool that runs the routing prompts against an endpoint and reports the numbers for the config.

The contract parameters (TBD, pending Pi benchmarks):
- **Max TTFT:** ___ ms
- **Min generation tok/s:** ___
- **Max p99 latency:** ___ ms
- **Availability:** must respond within timeout or be marked degraded

The system derives timeouts directly from these declared values. A routing call to a backend declaring `max_ttft_ms = 400` gets a 400ms timeout. No guessing.

### Contract Violation: Hard vs Soft Timeout

The performance contract enables two operation modes, configured per backend:

**Hard timeout** (`timeout_mode = "hard"`):
1. Routing call exceeds `max_p99_ms` → crash immediately
2. Log warning, mark backend `:degraded`
3. Retry on next healthy `:routing` backend
4. No healthy backends → panic, alert operator

Best for: deployments with redundant `:routing` backends. Fail fast, let the next one handle it.

**Soft timeout** (`timeout_mode = "soft"`):
1. Routing call exceeds `max_p99_ms` → log notice, keep waiting up to `3 × max_p99_ms`
2. Still no response → log warning, mark backend `:degraded`, try next `:routing` backend if available
3. No other backends → keep waiting (it's all you've got), escalate to operator notification
4. Eventually responds → log the slow response, process it, but flag the backend for health review

Best for: single-backend deployments where there's nowhere to fail over to. A Pi that thermal throttles sometimes is still better than nothing.

```toml
[performance]
max_ttft_ms = 400
min_tok_per_sec = 30
max_p99_ms = 3500
```

Which mode applies is determined by the system-wide `operation_mode` setting (see above), not per-backend. `"strict"` = hard timeout, `"loose"` = soft timeout. The contract parameters are the same either way — what changes is the consequence of violating them.

OTP supervision makes both modes safe. The contract gives us permission to be strict *and* the flexibility to be forgiving when the operator knows the tradeoffs.

## Structured Output Ownership

Myelin owns the canonical output schemas — not the backends.

The system needs structured responses for many workflows: routing decisions, tool calls, entity extraction, self-narrated context. These schemas define *what* the response looks like. The backend config declares *which format dialect* the model speaks.

### How it works

1. **`Myelin.Schemas`** (or similar) is a central repository of all output schemas
2. Each schema exists in multiple format dialects: `:gbnf`, `:json_schema`, potentially others
3. Backend config declares: `structured_output = "gbnf"` or `structured_output = "json_schema"`
4. At call preparation time: the function building the API request calls `Myelin.Schemas.get(:chat_response, :gbnf)` to get the right format for the target backend

### Why central ownership

- **Consistency:** all backends producing `:chat_response` output produce the *same structure*, just in different format dialects. The downstream code has one parser, not N.
- **Maintainability:** changing a schema is one place, not per-backend.
- **Extensibility:** adding support for a new format dialect (e.g., `:regex_grammar`) is: write the translations for existing schemas, declare it in backend configs, done. No per-model schema files.

### Structural requirements by capability

| Capability | Requires structured output? | Why |
|---|---|---|
| `:routing` | **Yes** — GBNF or JSON Schema | Routing decisions must be machine-parseable |
| `:chat` | **Yes** — GBNF or JSON Schema | Async tool calls + self-narrated context depend on structured responses |
| `:reasoning` | Preferred but not required | Structured output helps but free-form reasoning is sometimes better |
| `:summarization` | No | Plain text output is fine |
| `:feature_extraction` | **Yes** | Entity/tag output needs structure |
| `:vision` | No | Descriptions can be free-form |

## Operation Mode

One line in the system config sets the deployment posture:

```toml
# myelin.toml
operation_mode = "loose"    # or "strict"
```

This isn't a per-backend or per-feature toggle — it's a system-wide stance on how paranoid the deployment is. Every subsystem reads it and adjusts defaults accordingly.

| Concern | `"loose"` | `"strict"` |
|---|---|---|
| Routing timeout | Soft — be patient, you probably only have one backend | Hard — fail fast, fail over |
| Degradation alerts | Relaxed — longer grace periods before notification | Aggressive — notify immediately on degradation |
| Budget enforcement | Warn but don't gate — let calls through with warnings | Hard caps per capability class — reject when exhausted |
| Interface timeouts | Forgiving — sessions can wait | Strict — drop unresponsive connections |
| Startup validation | Warn on missing capabilities, start anyway | Refuse to start if critical capabilities are missing |
| Health probing | Less frequent, more tolerant of transient failures | Frequent, strict pass/fail against contract |

**`"loose"` is the default.** It's what you want when you're running Myelin on a Pi with one routing model and OpenRouter on the side. Things might be slow sometimes, and that's fine — you'd rather have a sluggish response than a crash and an alert at 2am.

**`"strict"` is for when it matters.** Multiple backends, redundancy, real uptime expectations. Fail fast, alert early, enforce budgets. The system has enough capacity to recover from crashes, so crashes are the right response to contract violations.

Individual settings can still be overridden — `operation_mode` just sets the defaults. Think of it as a preset that you can fine-tune.

## Panic States

If **all backends declaring `:routing`** are `:degraded` or `:offline`, that's a panic state. The system cannot route events and must surface this immediately via `:backend_critical` PubSub — and push a notification to the operator ("routing down, all `:routing` backends degraded/offline"). The operator needs to know *now*, not discover it later when they wonder why nothing's happening.

A system that stalls silently is worse than one that crashes loudly. The core must never quietly degrade into a state where events pile up with no routing — that just looks broken with no clue why.

Other capability losses are degraded but not panic. As long as routing is alive, the system can still triage: "this is important, queue it for when `:reasoning` comes back" vs "this is noise, drop it." The router is the last line of defense — even with zero other inference available, it keeps events from piling up unprocessed. It can't *act* on them, but it can make sure the important ones are waiting when capacity returns.

Only `:routing` loss is existential because the system literally cannot decide what to do with incoming events.

## Deployment Profiles

Same software, same capability graph. The cost/hardware tradeoff is the operator's choice.

**"Openclaw mode"** — zero hardware, maximum cloud:
- Everything on Anthropic/OpenRouter, including routing
- Works immediately, no setup beyond API keys
- Expensive (~$??/mo depending on volume), and your system dies if the internet does
- But it *works*, and you can migrate to local routing later without changing anything

**Headless monitor** — Pi + one model:
- 1x small local model: `:routing` + `:structured_output` + `:processing`
- Watches events, runs Processor recipes, no conversation
- Cost: electricity. That's it.

**Smart budget agent** — Pi + OpenRouter (~$30/mo estimate):
- 1x small local model: `:routing` + `:structured_output` (always-on, free)
- 1x OpenRouter cheap tier: `:summarization` + `:feature_extraction` + `:processing` (bulk work, pennies)
- 1x OpenRouter capable tier: `:chat` + `:reasoning` + `:tool_use` (sessions/engaged, most of the spend)
- Full-featured agent, local routing for privacy/reliability, remote for capability
- The sweet spot for someone who's smart about it

**Full local** — privacy maximalist:
- 1x small local model: `:routing` + `:structured_output`
- 1x large local model (24GB+ GPU): `:agent` + `:processing` + maybe `:vision`
- Zero external calls. Zero cloud dependency. Fully sovereign.
- Capability limited by your hardware, but it's *yours*

**Kitchen sink** — everything connected:
- Local routing + Anthropic for frontier reasoning + OpenRouter for bulk + local embeddings + vision model
- Maximum capability, optimized cost (cheap work goes cheap, expensive work goes to the best model)
- The "I want the best agent possible" configuration

The capability graph validates any of these at startup. Missing `:chat`? Interactive sessions disabled, everything else works. No `:embedding`? Keyword search only. The system tells you exactly what you get with what you've connected.

## OpenRouter Economics

Remote commodity models via OpenRouter make mechanical Processor work nearly free:
- ~1k calls at ~4k tok in / ~500 tok out ≈ $0.33
- Batch summarization, feature extraction, timeline building can run aggressively instead of conservatively
- Budget becomes per-capability-class, not a single global gate

## Backend Selection

When multiple backends declare the same capability, BackendPool picks one. Default priority:

**health > cost > latency**

That's the sane default. If you want to tweak it — `BackendPool.set_selection_priority([:health, :latency, :cost])` from an IEx shell. No config UI, no admin panel. Power users who care about selection order can drop into Elixir. Everyone else gets the default.

## Backend Registration

Backends are defined as **TOML config files** — one per backend, in `config/backends/`. Human-readable, version-controllable, validated on load.

### Config file schema

Full annotated example showing every section:

```toml
# config/backends/anthropic-sonnet.toml

# ── Identity ──────────────────────────────────────────────
name = "anthropic-sonnet"            # unique, used in logs/status/selection
model = "claude-sonnet-4-6"          # model ID passed in API calls
description = "Frontier reasoning"   # optional, for operator reference

# ── Connection ────────────────────────────────────────────
endpoint = "https://api.anthropic.com/v1"
api_key_env = "ANTHROPIC_API_KEY"    # env var name — never inline secrets
calling_convention = "anthropic"     # "openai" | "anthropic"

# Optional: token counter endpoint for prompt pre-flight sizing
# If absent, system estimates from tokenizer heuristics
token_counter = "https://api.anthropic.com/v1/messages/count_tokens"

# ── Capabilities ──────────────────────────────────────────
capabilities = ["agent", "reasoning", "chat", "tool_use"]
structured_output = "json_schema"    # "gbnf" | "json_schema" | "none"

# ── Model Limits ──────────────────────────────────────────
context_window = 200_000             # max input tokens
max_output_tokens = 8192             # max generation length

# ── Cost (USD per million tokens) ─────────────────────────
# Required for remote backends. For local backends, omit or set to 0.
# Use the most expensive tier if the provider has variable pricing —
# ballpark is fine for now, exact pricing integration is future work.
[cost]
input_per_mtok = 3.00
output_per_mtok = 15.00
cache_write_per_mtok = 3.75          # cost to write into prompt cache
cache_read_per_mtok = 0.30           # cost to read from prompt cache

# ── Cache ─────────────────────────────────────────────────
# Prompt caching configuration. Omit entire section if backend
# doesn't support prompt caching.
[cache]
supported = true
marker_style = "anthropic"           # "anthropic" (cache_control blocks)
                                     # | "openai" (prefix-based, automatic)
                                     # | "none"
# TTL tiers — Anthropic has 5min ephemeral and 1h standard.
# Declare what the backend supports; Myelin picks the best tier per use case.
ttl_tiers = [
  { name = "ephemeral", seconds = 300 },
  { name = "standard", seconds = 3600 },
]

# ── Performance (routing backends only) ───────────────────
# REQUIRED if capabilities include "routing".
# Values should come from actual benchmarks — Myelin can ship
# with a benchmarking tool to populate these.
# [performance]
# max_ttft_ms = 400
# min_tok_per_sec = 30
# max_p99_ms = 3500
```

### Minimal examples for common setups

```toml
# config/backends/local-llama.toml — Pi routing model
name = "local-llama"
model = "qwen2.5-7b-instruct"
endpoint = "http://localhost:8080/v1"
calling_convention = "openai"
token_counter = "http://localhost:8080/tokenize"
capabilities = ["routing", "structured_output", "processing"]
structured_output = "gbnf"
context_window = 8192
max_output_tokens = 2048

[performance]
max_ttft_ms = 400
min_tok_per_sec = 30
max_p99_ms = 3500
```

```toml
# config/backends/openrouter-bulk.toml — cheap processing
name = "openrouter-bulk"
model = "meta-llama/llama-3.1-8b-instruct"
endpoint = "https://openrouter.ai/api/v1"
api_key_env = "OPENROUTER_API_KEY"
calling_convention = "openai"
capabilities = ["summarization", "feature_extraction", "processing"]
structured_output = "json_schema"
context_window = 131072
max_output_tokens = 4096

[cost]
input_per_mtok = 0.06
output_per_mtok = 0.06
cache_write_per_mtok = 0.0
cache_read_per_mtok = 0.0
```

### Schema reference

**Top-level fields:**

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | Yes | string | Unique across all backend configs |
| `model` | Yes | string | Model ID for API calls |
| `description` | No | string | Operator notes |
| `endpoint` | Yes | string | Base URL |
| `api_key_env` | For remote | string | Env var name containing API key |
| `calling_convention` | Yes | `"openai"` \| `"anthropic"` | Determines request/response format |
| `token_counter` | No | string | URL for token counting endpoint (pre-flight) |
| `capabilities` | Yes | string[] | Primitive or composite capability names |
| `structured_output` | If needed | `"gbnf"` \| `"json_schema"` \| `"none"` | Format dialect for structured generation |
| `context_window` | Yes | integer | Max input tokens |
| `max_output_tokens` | Yes | integer | Max generation length |

**`[cost]` section** (required for remote, omit for local):

| Field | Type | Notes |
|---|---|---|
| `input_per_mtok` | float | USD per million input tokens |
| `output_per_mtok` | float | USD per million output tokens |
| `cache_write_per_mtok` | float | USD per million tokens written to cache |
| `cache_read_per_mtok` | float | USD per million cached tokens read |

**`[cache]` section** (omit if no prompt caching):

| Field | Type | Notes |
|---|---|---|
| `supported` | bool | Explicit opt-in |
| `marker_style` | `"anthropic"` \| `"openai"` \| `"none"` | How cache markers are expressed in API calls |
| `ttl_tiers` | array of `{name, seconds}` | Available cache durations |

**`[performance]` section** (required if `:routing` in capabilities):

| Field | Type | Notes |
|---|---|---|
| `max_ttft_ms` | integer | Max time-to-first-token in ms |
| `min_tok_per_sec` | integer | Minimum generation throughput |
| `max_p99_ms` | integer | Hard ceiling for total call duration |

### Validation rules

On load, the system validates:
1. All required fields present and correctly typed
2. `name` is unique across all backend configs
3. Capabilities are known primitives or composites
4. If `:routing` declared → `[performance]` block exists with all required fields
5. If capability requires structured output → `structured_output` is declared and not `"none"`
6. If `api_key_env` declared → the env var exists and is non-empty
7. `calling_convention` is a known value
8. `context_window` and `max_output_tokens` are positive integers
9. Cost values are non-negative if `[cost]` is present
10. `[cache].ttl_tiers` entries have positive `seconds` values

Invalid configs are rejected with clear error messages naming the file, field, and problem. The system starts with whatever valid backends it has.

### Runtime

`BackendPool.reload_backends()` reloads all configs at runtime. Hot-reload, no restart needed.

**The agent cannot modify backend configs.** This is operator territory — adding/removing inference backends is an infrastructure decision, not something the system should do autonomously. The agent can *read* backend status and *report* problems, but never touch the config.

## Degradation & Recovery

Not all failures are equal. The recovery strategy depends on *why* a backend went down:

### Server errors (5xx, connection refused, timeouts)

The endpoint is probably having a bad time. Poll every **10 minutes** to see if it comes back. If it responds healthy, promote back to `:online`. Don't invest heavily — it'll either recover or it won't.

### Performance degradation (contract violations)

The backend is responding but too slowly. This is more concerning — it might be overloaded, thermally throttling, or sharing resources. Mark `:degraded`, **notify the operator**. Don't auto-promote back to healthy without a successful probe that meets the contract parameters.

### Auth/permission errors (403, 401)

Something is wrong with credentials or access. **Notify the operator immediately** — this won't fix itself. Don't waste probe cycles on it.

### Extended degradation

Any backend that's been `:degraded` or `:offline` for **>2 hours** triggers a **mandatory operator notification**, regardless of cause. If you haven't noticed by now, the system makes sure you do.

### Summary

| Failure type | Action | Probe interval | Auto-recovery? | Operator notification? |
|---|---|---|---|---|
| 5xx / connection error | Mark `:offline`, poll | 10 min | Yes, on healthy probe | No (unless >2h) |
| Contract violation (slow) | Mark `:degraded` | 10 min, must pass contract | Yes, on passing probe | **Yes, immediately** |
| 403/401 | Mark `:offline` | None | No | **Yes, immediately** |
| Any state >2h | — | — | — | **Mandatory** |

## Budget & Routing Exemption

Budget tracks spend per capability class. But `:routing` is exempt from budget constraints entirely.

**Why:** Myelin guarantees that routing prompts are small — a handful of tokens in, a handful out. The cost is negligible regardless of backend. More importantly, routing is the one thing that must *never* be throttled. If Budget could gate routing, you'd get a death spiral: system runs out of budget → can't route → can't decide what to do → events pile up → operator sees a dead system.

`:routing` always fires. Everything else is subject to per-class budgets:
- `:summarization` / `:feature_extraction` / `:processing` — cheap, high allowance
- `:reasoning` / `:interactive` — expensive, tracked carefully

## Architectural Implications

1. **BackendPool** manages backends with declared capabilities, selects by health > cost > latency
2. **Processor recipes** specify required capability, not a specific backend
3. **InferenceRouter** always dispatches routing work to a `:routing`-capable backend — no budget gate
4. **Budget** tracks spend per capability class, `:routing` exempt
5. **Backend configs** are operator-managed files, not agent-modifiable
6. Multiple backends can declare the same capability — BackendPool picks based on selection priority

## Current State

Today we have:
- `LlamaClient` — local llama.cpp, handles routing (implicitly `:routing` capable)
- `AnthropicClient` — remote, handles Interactive/Engaged/Processor (implicitly `:reasoning` + `:interactive` + `:processing`)

The refactor (#39) will make capabilities explicit and allow N backends. Until then, this document captures the design direction so nothing we build assumes a single dedicated processor or hardcodes backend selection.
