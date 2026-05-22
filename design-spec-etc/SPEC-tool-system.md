# Tool System

*Two grammars for two timescales. Sync tools read the environment before responding. Async tools change the world and report back later.*

---

## The Split

The fundamental distinction is **what the model needs the result for**:

| Category | Purpose | Timescale | Mechanism |
|---|---|---|---|
| **Sync tools** | Read environment to inform *this* response | In-turn, blocking | Declared to API, handled by tool_use loop |
| **Async tools** | Adjust information environment for *future* turns | Out-of-turn, fire-and-forget | Declared in system prompt, dispatched via structured output |

**Sync tools answer questions. Async tools change the world.**

The API's native tool_use infrastructure is optimized for sync tools — it stops generation, waits for results, resumes. That's exactly right for "look up this entity before drafting a response." It's completely wrong for "go search the web while I keep talking."

---

## Sync Tools (API-Declared)

Declared in the frozen zone as standard API tool schemas. The model calls them, generation pauses, result is injected, generation resumes. Blocking is correct — the response depends on the result.

### Canonical sync tools

```elixir
# Memory and context reads — "what do I know about this?"
engram_search(query: String.t(), top_k: integer())
  :: [%{text: String.t(), similarity: float()}]

entity_lookup(handle: String.t())
  :: %{tier: integer(), context: String.t(), recent_interactions: [...]} | nil

system_state(component: atom())
  :: map()  # current state of any registry/component

hot_topic_check(text: String.t())
  :: %{match: boolean(), topics: [String.t()], score: float()}

conversation_context(conversation_id: String.t())
  :: %ConversationContext{} | nil
```

### SSH sessions (sync, special case)

SSH sessions are sync because **the model IS the session**. A worker model running an SSH session has nothing else to do — it's sitting at a terminal, reading output, deciding what to type next. Blocking is correct. The worker's entire purpose is this terminal session.

```elixir
# Sync — worker waits for command output before deciding next step
ssh_exec(host: String.t(), command: String.t(), timeout_s: integer())
  :: {:ok, output: String.t(), exit_code: integer()} | {:error, reason: String.t()}

# Not fire-and-forget — the model reads the output and continues the session
```

This is distinct from `ssh_task` (async) — see below.

### Placement in frozen zone

```
Layer 1 (Situational, 1-hour cache):
  [Rules snapshot]
  [Entity registry]
  [Tool declarations — sync tools]    ← stable, cached with Layer 1
```

Sync tool schemas are stable — they don't change turn-to-turn, so they earn their place in the 1-hour cached layer.

---

## Async Tools (Structured Output)

Declared in the system prompt (Layer 2). The model issues async tool calls via the `tool_calls` field in the structured output response. The system parses and dispatches them. Results arrive later as events in the originating conversation.

### Canonical async tools

```elixir
# Web and platform fetches — slow, network-bound
web_search(query: String.t(), max_results: integer())
fetch_thread(uri: String.t(), depth: :shallow | :full)
bluesky_search(query: String.t(), filters: map())

# Task delegation — fire and observe
ssh_task(host: String.t(), playbook: String.t(), args: map())
  # Fires a pre-defined script or Ansible playbook at a host.
  # Does NOT give the model interactive terminal access.
  # Use ssh_exec (sync) for interactive sessions.

schedule_task(op: atom(), args: map(), delay_s: integer())
write_rule(rule: map())
enrich_memories(keywords: [String.t()], lookback_days: integer())

# Synthesis — longer-running processor calls
summarize_with_instruct(content: String.t(), instruct: String.t(), max_tokens: integer())
fetch_and_summarize(uri: String.t(), focus: String.t())
```

### Dispatch format (in structured output)

```json
{
  "tool_calls": [
    {
      "op": "web_search",
      "args": { "query": "elixir OTP supervision patterns", "max_results": 10 },
      "ref": "search-abc123",
      "failure_mode": "next_turn"
    },
    {
      "op": "write_rule",
      "args": { "trigger": "mention:atproto", "action": "boost_salience", "value": 0.2 },
      "ref": "rule-xyz456",
      "failure_mode": "inject"
    }
  ]
}
```

The `ref` is model-generated (or system-assigned if omitted) — a short unique identifier that tracks this call through its lifecycle.

---

## Failure Modes

Every async tool call declares how failures should be handled. The model decides on the spot based on how critical the call is.

### Three modes

| Mode | Behavior | When to use |
|---|---|---|
| `"silent"` | Log to failure queue, no session notification | Best-effort enrichment, redundant lookups, speculative work where failure is expected occasionally |
| `"next_turn"` | Note in next natural turn's hot zone | Most async work — "by the way, search-abc123 failed" when we naturally check in |
| `"inject"` | Interrupt immediately with a new turn | Critical path failures, security events, anything where the model needs to decide right now |

### Optimistic by default

The system assumes success. No "awaiting confirmation" machinery needed for happy path:

```
turn N:   model fires web_search (ref: search-abc123, failure_mode: next_turn)
turn N+1: hot zone includes "search-abc123 completed: [result summary]"
           OR: model just continues if result isn't ready yet
turn N+k: if failure — hot zone includes "search-abc123 FAILED: connection timeout"
```

The model only hears about problems. Successes are folded into the next natural turn's context.

---

## Failure Queue

Everything that fails gets written to a `tool_failures` table regardless of failure_mode. The failure_mode only controls *when the model hears about it* — the queue captures everything for operational review.

```sql
CREATE TABLE tool_failures (
  id TEXT PRIMARY KEY,                -- ulid
  ref TEXT NOT NULL,                  -- the tool call ref
  op TEXT NOT NULL,                   -- which tool
  args TEXT NOT NULL,                 -- JSON
  error_type TEXT NOT NULL,           -- :timeout | :network | :auth | :invalid_args | ...
  error_message TEXT,
  conversation_id TEXT,               -- originating conversation
  session_id TEXT,                    -- originating session
  failure_mode TEXT NOT NULL,         -- how we handled it
  notified_at TEXT,                   -- when we told the model (if we did)
  resolved_at TEXT,                   -- if manually resolved/retried
  created_at TEXT NOT NULL
);
```

### Failure queue maintenance

Reviewed on the 1h maintenance cycle. Two-phase process: mechanical counting first, then Processor evaluation only if something stands out.

**Phase 1 — mechanical counting (free):**

```elixir
defmodule Myelin.FailureQueue.Evaluator do
  def evaluate(window_hours \\ 12) do
    since = DateTime.add(DateTime.utc_now(), -window_hours * 3600, :second)

    failures = FailureQueue.fetch_since(since)
    total    = ToolCallLog.count_since(since)

    by_op    = Enum.group_by(failures, & &1.op)
    rate     = length(failures) / max(total, 1)

    %{
      total_jobs: total,
      total_failures: length(failures),
      failure_rate: rate,
      by_op: Map.new(by_op, fn {op, fs} -> {op, length(fs)} end)
    }
  end
end
```

**Phase 2 — Processor evaluation (only if failure_rate > threshold):**

If the mechanical pass flags something interesting (failure rate above ~15%, or a single op with 3+ failures), pass a brief summary to the Processor for classification:

```
You are a systems reliability evaluator. Over the last 12 hours:
3 jobs of the type "web_fetch" have failed,
2 jobs of the type "rule_add" have failed,
1 job of the type "entity_lookup" failed.
A total of 18 jobs were run, 6 of which failed.

Respond with exactly one word: normal, review, or warning.
```

**Response → action mapping:**

| Verdict | Action |
|---|---|
| `normal` | Log, continue. Nothing to surface. |
| `review` | Add a note to the next Interactive session's hot zone: "failure queue has items worth a look." Human decides if it matters. |
| `warning` | Inject immediately into any active Interactive session. Something is actually wrong. |

The prompt is intentionally minimal — the Processor isn't asked to diagnose, explain, or recommend. It issues a verdict. One word. The human (or a future escalation path) does the rest.

- **Retry eligibility:** After verdict, transient failures (timeout, network) older than 5min flagged as eligible for retry dispatch
- **Pruning:** Resolved failures older than 7 days → archive

The failure queue is the operational heartbeat: "here's everything the system tried and couldn't do." Grooming it regularly catches infrastructure problems before they become incidents.

---

## Result Delivery

When an async tool call completes, the result is an event in the originating conversation:

```elixir
%Event{
  source: :internal,
  kind: :tool_result,
  signals: %{
    thread_id: conversation_id,   # routes to originating conversation
    tool_ref: "search-abc123",
    tool_op: :web_search,
    success: true
  },
  summary: "web_search completed: 8 results for 'elixir OTP patterns',
            top results: [brief descriptions]"
}
```

For `:inject` failures, the EventPipeline routes this as a high-priority event that creates a synthetic turn in the Interactive session immediately. For `:next_turn`, it's folded into the hot zone at the next natural call. For `:silent`, it goes to the failure queue only.

---

## Decision Guide

**Use sync (API tool):**
- You need the result before you can write the response
- The call completes in <500ms (memory lookups, local state reads)
- You're in a worker SSH session and need terminal output to decide next command

**Use async (structured output):**
- Result feeds a future turn, not this one
- Call has unbounded latency (network, remote execution)
- You're delegating work to happen in parallel while the conversation continues
- You want to fire multiple independent operations simultaneously

**Use ssh_exec (sync SSH):**
- Worker model running an interactive diagnostic session
- Need to read output before deciding what to run next
- Model IS the operator

**Use ssh_task (async):**
- Running a known playbook or script that doesn't need interaction
- Model wants to kick off work and hear the result later
- Safe to fire and move on

---

## Design Properties

- **Unified observability.** Both sync and async tool calls produce events in the conversation. The dashboard shows everything — what was called, when, with what args, what returned. The API's native tool_use is a black box; ours isn't.

- **Model decides failure handling.** The model knows at call time how critical a given tool call is. It sets the failure_mode accordingly. No external policy needed.

- **Failure queue is always there.** Regardless of failure_mode, failures are captured. The queue is for operational review — catching infrastructure problems, detecting patterns, enabling retries.

- **Optimistic by default.** Happy path is silent. The model focuses on what it's trying to do, not on confirming that background work is proceeding.

- **SSH session vs SSH task is a clear semantic distinction.** Interactive terminal = sync. Fire-and-observe = async. The naming makes the intent explicit.
