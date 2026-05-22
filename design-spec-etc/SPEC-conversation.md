# Conversations — The Missing First-Class Citizen

*A conversation is any stateful multi-turn exchange, regardless of medium. Bluesky thread, Telegram chat, bash session, tool call chain — same shape, same type, same routing affordances.*

---

## The Insight

Events arrive individually. But most meaningful things happen in sequences: a thread develops, a discussion unfolds, a tool session iterates toward a result. The system currently handles this implicitly — `signals.thread_id` exists on events, sessions track `primary_thread_id` — but there's no object that represents the conversation itself. No place to accumulate understanding of what a thread is *about*, who's involved, how salient it's been over time, or what the agent's relationship to it is.

Without conversations as a first-class type:
- The router re-evaluates every event from scratch, even if it's the 15th reply in a thread we're already tracking
- Salience can't compound — a thread that's been interesting for an hour doesn't get credit for that history
- Tool call chains are stateless — each step is a new event with no memory of the chain
- The system can't distinguish "new noise from a stranger" from "continuation of something we care about"

---

## Design Principles

### Stamp at the edge, reason in the center

Interfaces are stateless. They stamp each incoming event with whatever natural conversation identifier the platform provides. That's it. No tracking, no state, no intelligence about what conversations mean.

| Platform | Natural conversation_id | Notes |
|---|---|---|
| Bluesky | `at://` root post URI | Thread root, available on every reply |
| Telegram | `chat_id` | Persistent per chat/group |
| Discord | `channel_id` | Or thread_id for threaded channels |
| IRC/Matrix | room ID | |
| Syslog | host + service tuple | Grouping by origin |
| Bash/TTY | PTS path or session ID | Each terminal is a conversation |
| Tool chain | Internally generated chain_id | Myelin creates these |
| Internal | Event type grouping | state_changed, cache_available, etc. |

The interface doesn't know if a conversation is important, tracked, or new. It stamps and emits. The EventPipeline enrichment step does the lookup:

```
incoming event (stamped with conversation_id by interface)
  → EventPipeline.enrich()
  → ConversationRegistry.lookup(conversation_id)
  → if known: inject conversation context into event signals
  → if unknown: proceed normally, maybe open new conversation record later
```

### Crash resilience as a design property

All conversation state lives inside the runtime (ConversationRegistry). If an interface crashes and reconnects, it starts stamping events again. The inside picks up exactly where it left off. No state lost. This is OTP supervision doing what it was built for.

### Conversations are not sessions

These are related but distinct concepts:

| | Conversation | Session |
|---|---|---|
| **What** | An ongoing exchange | "The system did a thing" |
| **Origin** | External (platform activity) | Internal (state machine escalation) |
| **Concurrency** | Many concurrent | One active at a time |
| **Lifecycle** | Tied to platform activity | Tied to state machine |
| **Duration** | Minutes to weeks | Minutes to ~90min (hard TTL) |
| **Persistence** | Long-lived, decaying | Closed → compacted → pruned |

A session might process events from multiple conversations. A conversation might span multiple sessions. The relationship is many-to-many but typically clustered: a session usually has one primary conversation that triggered escalation, with others injected or deferred.

---

## Schema

```elixir
defmodule Myelin.Conversation do
  @type t :: %__MODULE__{
    # identity
    id: String.t(),                      # normalized conversation_id from platform
    source: atom(),                      # :bluesky | :telegram | :bash | :tool_chain | :internal

    # participants
    participants: [%{
      handle: String.t(),
      role: atom(),                      # :initiator | :participant | :agent | :system
      first_seen: DateTime.t(),
      last_seen: DateTime.t(),
      message_count: non_neg_integer()
    }],

    # topology
    root_event_id: String.t() | nil,     # first event in the conversation
    branch_count: non_neg_integer(),     # for tree-structured threads (Bluesky)
    depth: non_neg_integer(),            # max reply depth
    turn_count: non_neg_integer(),       # total events in this conversation
    our_turn_count: non_neg_integer(),   # events where the agent participated

    # classification (evolves over time)
    topic_signature: [float()] | nil,    # embedding of conversation summary
    topic_keywords: [String.t()],        # extracted from content over time
    conversation_type: atom() | nil,     # see Conversation Type Vocabulary below
    sentiment: atom() | nil,             # :positive | :neutral | :negative | :heated

    # salience history
    cumulative_salience: float(),         # rolling weighted sum of event saliences
    peak_salience: float(),              # highest single-event salience
    salience_trend: atom(),              # :rising | :stable | :declining | :dead
    last_salience: float(),              # most recent event's salience

    # agent relationship
    tracking_level: atom(),              # :untracked | :watched | :active | :interactive
    policy_overrides: map(),             # conversation-level routing overrides
    last_agent_action: DateTime.t() | nil, # when we last did something in this conversation
    agent_role: atom() | nil,            # :observer | :participant | :driver

    # lifecycle
    state: atom(),                       # :open | :idle | :closed | :archived
    opened_at: DateTime.t(),
    last_activity: DateTime.t(),
    idle_since: DateTime.t() | nil,      # set when no events for decay_threshold
    closed_at: DateTime.t() | nil,

    # summary (maintained by Processor)
    summary: String.t() | nil,           # current understanding of conversation content
    summary_updated_at: DateTime.t() | nil,
    summary_covers_turn: non_neg_integer() # which turn the summary is current through
  }
end
```

---

## ConversationRegistry

A GenServer managing all known conversations. Backed by ETS for fast lookups (every event enrichment does a lookup), SQLite for persistence.

```elixir
defmodule Myelin.ConversationRegistry do
  @moduledoc """
  Manages the lifecycle and state of all known conversations.
  ETS for hot reads (event enrichment path), SQLite for persistence.
  """

  # --- Hot path (called during event enrichment, must be fast) ---

  def lookup(conversation_id) :: %Conversation{} | nil
  # ETS read. Microseconds. Called on every single inbound event.

  def touch(conversation_id, %Event{}) :: :ok
  # Update last_activity, increment turn_count, update participant last_seen.
  # ETS write + async SQLite persist.

  # --- Warm path (called during routing/scoring) ---

  def get_context(conversation_id) :: %ConversationContext{}
  # Returns the subset of conversation data useful for salience scoring
  # and router context assembly. Struct, not full conversation.

  def conversations_for_entity(handle) :: [%Conversation{}]
  # "What conversations is this person in?" For entity-aware routing.

  def active_conversations() :: [%Conversation{}]
  # All conversations with tracking_level in [:watched, :active, :interactive]

  # --- Cold path (management, not on event hot path) ---

  def open(conversation_id, source, root_event) :: %Conversation{}
  def close(conversation_id, reason) :: :ok
  def archive(conversation_id) :: :ok
  def set_tracking_level(conversation_id, level, opts \\ []) :: :ok
  def set_policy_override(conversation_id, key, value) :: :ok
  def update_summary(conversation_id, summary, covers_turn) :: :ok
  def update_classification(conversation_id, attrs) :: :ok

  # --- Maintenance ---

  def decay_idle(threshold_minutes) :: non_neg_integer()
  # Move conversations with no activity past threshold to :idle.
  # Returns count of conversations idled.

  def prune_closed(older_than_days) :: non_neg_integer()
  # Archive closed conversations older than threshold.
end
```

### ConversationContext — the lightweight routing struct

The full `%Conversation{}` is too heavy for the event enrichment path. The router needs a thin struct:

```elixir
defmodule Myelin.ConversationContext do
  @type t :: %__MODULE__{
    id: String.t(),
    tracking_level: atom(),
    cumulative_salience: float(),
    salience_trend: atom(),
    conversation_type: atom() | nil,
    turn_count: non_neg_integer(),
    our_turn_count: non_neg_integer(),
    policy_overrides: map(),
    last_activity: DateTime.t(),
    topic_keywords: [String.t()]
  }
end
```

This is what gets injected into event signals during enrichment. ~200 bytes, sits in ETS alongside the full conversation.

---

## Conversation Lifecycle

```
                    first event with unknown conversation_id
                                    │
                                    ▼
                            ┌──────────────┐
                            │   :open       │ ← created, tracking_level: :untracked
                            │               │   most conversations stay here briefly and die
                            └───────┬───────┘
                                    │
                        ┌───────────┼──────────────────┐
                        │           │                   │
                    (silence)   (continued          (agent engages or
                        │        activity)           manual watch)
                        ▼           │                   │
                ┌──────────────┐    │           ┌───────┴───────┐
                │   :idle       │    │           │   :open       │
                │               │    │           │   :watched+   │ ← tracking_level promoted
                └───────┬───────┘    │           └───────────────┘
                        │           │
                    (more silence)  │
                        │           │
                        ▼           │
                ┌──────────────┐    │
                │   :closed     │◄──┘ (or: TTL expires on untracked conversations)
                │               │
                └───────┬───────┘
                        │
                  (older than N days)
                        │
                        ▼
                ┌──────────────┐
                │   :archived   │ ← summary preserved in engram, raw data prunable
                └──────────────┘
```

### Tracking levels

| Level | Meaning | Router behavior | Opened by |
|---|---|---|---|
| `:untracked` | We know about it, don't care yet | Normal scoring, no boost | Default for all new conversations |
| `:watched` | Passively monitoring | Salience boost (1.3x), events added to conversation record | Router escalation, rule (`boost_thread`), agent decision |
| `:active` | We're participating | Larger boost (1.6x), summary kept current, Processor tasks scheduled | Agent posts/replies in conversation |
| `:interactive` | Human-driven live session | Router mostly bypassed, direct to Engaged/Interactive | `:human_connect` event, Telegram session open |

Tracking level can be set by:
- **Router:** escalation on a conversation event promotes `:untracked` → `:watched`
- **Rules:** `boost_thread(conversation_id, ...)` promotes to `:watched` or `:active`
- **Agent decision:** Sonnet decides to participate, promotes to `:active`
- **Human event:** `:human_connect` promotes to `:interactive`
- **Decay:** no activity for threshold → demote one level

### Tracking level decay

```
:interactive  ──(30min silence)──→  :active
:active       ──(2h silence)────→  :watched
:watched      ──(6h silence)────→  :untracked
:untracked    ──(24h silence)───→  :closed (if turn_count < 3)
                                   :idle (if turn_count >= 3)
```

These are defaults. Conversation-level policy overrides can extend or shorten them. A `pin_state` rule on a conversation keeps its tracking level from decaying.

---

## Integration with Event Enrichment

The EventPipeline enrichment step gains a conversation lookup:

```elixir
defmodule Myelin.EventEnricher do
  def enrich(%Event{} = event) do
    # existing enrichment (parallel)
    entity_task = Task.async(fn -> MemoryStore.lookup_entity(event.sender) end)
    embed_task = Task.async(fn -> EmbeddingClient.embed(event.summary) end)
    keyword_task = Task.async(fn -> MemoryStore.match_keywords(event.summary) end)

    # NEW: conversation lookup (also parallel, also microseconds)
    conversation_task = Task.async(fn ->
      case event.signals.thread_id do
        nil -> nil
        conv_id ->
          ConversationRegistry.touch(conv_id, event)
          ConversationRegistry.get_context(conv_id)
      end
    end)

    entity = Task.await(entity_task)
    {hot_topic_score, embedding} = Task.await(embed_task)
    keyword_hits = Task.await(keyword_task)
    conversation_ctx = Task.await(conversation_task)

    %{event | signals: Map.merge(event.signals, %{
      hot_topic_match: hot_topic_score,
      entity_salience: entity && entity.salience,
      keyword_hits: keyword_hits,
      conversation: conversation_ctx       # NEW — nil if no thread_id or unknown conversation
    })}
  end
end
```

**Event schema addition:** `signals` gains a `conversation` field:

```elixir
signals: %{
  thread_id: String.t() | nil,
  parent_id: String.t() | nil,
  hot_topic_match: float() | nil,
  entity_salience: float() | nil,
  keyword_hits: [atom()],
  conversation: %ConversationContext{} | nil   # NEW
}
```

---

## Integration with Salience Scoring

Conversation context feeds directly into salience computation:

```elixir
defmodule Myelin.Salience do
  def score(%Event{} = event, %Policy{} = policy) do
    base = sender_tier_score(event)
         + hot_topic_component(event)
         + entity_component(event)
         + keyword_component(event)
         + conversation_component(event)     # NEW

    multiplier = kind_multiplier(event.kind)
    raw = min(base * multiplier, 1.0)
    apply_rules(raw, event)
  end

  # NEW
  defp conversation_component(%Event{signals: %{conversation: nil}}), do: 0.0
  defp conversation_component(%Event{signals: %{conversation: ctx}}) do
    tracking_boost = case ctx.tracking_level do
      :untracked -> 0.0
      :watched -> 0.1
      :active -> 0.2
      :interactive -> 0.3
    end

    trend_boost = case ctx.salience_trend do
      :rising -> 0.1
      :stable -> 0.0
      :declining -> -0.05
      :dead -> -0.1
    end

    # Conversations we're already in get a participation boost
    participation_boost = if ctx.our_turn_count > 0, do: 0.1, else: 0.0

    min(tracking_boost + trend_boost + participation_boost, 0.4)  # capped
  end
end
```

This means: a reply in a `:watched` thread that's trending upward gets +0.2 salience before any rules are applied. A reply in a thread we're actively participating in gets +0.3. A random event from an untracked, declining thread gets -0.05. The conversation's history directly shapes routing decisions.

---

## Integration with Router

The router gains conversation awareness in the two-pass flow:

### Pass 1 (vibe check)
Conversation context injected into the router prompt:
```
Event: reply from umbra.blue: "the discontinuity paper is relevant here"
Sender: tier-1
Conversation: tracked (watched), 12 turns, rising salience, topic: "identity/continuity"
→ vibe: notable
```

The router doesn't need to re-derive "this is an interesting thread" — the conversation context already encodes that signal.

### Pass 2 (routing decision)
For events in known conversations, the router gets conversation type:
```
Conversation type: :technical
Our participation: 3 turns (last: 2h ago)
Tracking: :active
→ action: process (fold into existing thread context)
```

### Router bypass for interactive conversations

When `tracking_level == :interactive`, the router is largely bypassed. Events flow directly to the active Interactive session. The StateMachine already has `force_interactive/0` — this is triggered automatically when a conversation transitions to `:interactive`.

```elixir
defp route_event(%Event{signals: %{conversation: %{tracking_level: :interactive}}} = event) do
  # Skip two-pass routing, direct to Interactive
  Interactive.inject_event(event)
end
```

---

## Conversation-Level Policies

Conversations can carry policy overrides that alter routing behavior:

```elixir
%{
  # Router bypass — events from this conversation skip the router
  bypass_router: true,

  # Minimum tracking level — decay won't drop below this
  min_tracking_level: :watched,

  # Custom event floor — override the state machine's policy
  event_floor: 0.2,

  # Priority delivery — outputs to this conversation get :urgent
  output_priority: :urgent,

  # Suppress — temporarily ignore this conversation
  suppress_until: ~U[2026-03-16 18:00:00Z]
}
```

These are set by:
- **Rules:** `boost_thread` / `suppress_thread` rules write to conversation policy overrides
- **Agent decision:** Sonnet can set policies via tool calls during Engaged sessions
- **Human:** Interactive mode can configure conversation policies manually

---

## Multi-Turn Tool Sessions

This is where conversations unlock a qualitatively new capability. A tool call chain wrapped in a conversation:

```elixir
# Agent decides to research something
chain_id = Myelin.Conversation.generate_chain_id()

ConversationRegistry.open(chain_id, :tool_chain, root_event)
ConversationRegistry.set_tracking_level(chain_id, :active,
  policy_overrides: %{
    bypass_router: true,       # tool returns don't need routing
    output_priority: :normal
  }
)

# Step 1: search
EventPipeline.ingest(%Event{
  source: :internal,
  kind: :tool_call,
  signals: %{thread_id: chain_id},
  summary: "web_search: elixir OTP supervision patterns"
})

# ... search returns, stamped with same chain_id ...
# Step 2: fetch details on interesting result
# Step 3: summarize with focus
# Step 4: enrich with memories
# Each step is an event in the same conversation
```

The conversation record accumulates:
- What tools were called and in what order
- What intermediate results looked like
- Topic drift over the chain (did the research veer off course?)
- Total cost of the chain
- Final result quality (was the output useful?)

This is a **research session** — and it's the same data type as a Bluesky thread. The Processor's `summarize_conversation` operation works on both.

### Task graph as conversation

The composable task graph we discussed earlier maps directly:

```elixir
%TaskGraph{
  conversation_id: chain_id,
  steps: [
    %{op: :fetch_thread, args: %{uri: "at://...", depth: :full}},
    %{op: :summarize_with_instruct, args: %{instruct: "focus on A, B, C", max_tokens: 500},
      depends_on: [0]},                  # depends on fetch_thread result
    %{op: :enrich_memories, args: %{keywords: ["Y", "Z"], lookback_days: 7}},
    %{op: :synthesize, args: %{}, depends_on: [1, 2]}  # merge summarize + enrich results
  ],
  delivery: :async,
  on_complete: :escalate_to_engaged
}
```

Each step emits events into the conversation. The conversation accumulates the results. The final step's event triggers escalation with the full conversation context available.

---

## Conversations as Composition

The conversation type is not just a data structure — it's an implicit task composition language. No DSL, no embedded interpreter, no external orchestrator. The conversation *is* the program.

### What you get for free

A `:tool_chain` conversation already provides the primitives of a composition system:

| Primitive | How conversations provide it |
|---|---|
| **Sequencing** | Events in a conversation are ordered by timestamp |
| **Dependency** | `TaskGraph.depends_on` gates steps on previous results |
| **Branching** | Processor evaluates intermediate results and decides the next step |
| **Error handling** | A failed tool call is an event with a failure signal — the Processor sees it and adapts |
| **State accumulation** | The conversation record accumulates everything — `extract_conversation_state` gives the current picture |
| **Resumability** | Conversation persists in SQLite — if the system crashes mid-chain, it picks up where it left off |
| **Observability** | Every step is an event, visible in the dashboard, auditable after the fact |

This is not a metaphor. A conversation with `source: :tool_chain` that runs fetch → summarize → enrich → synthesize *is* a program whose execution trace *is* the conversation record and whose interpreter *is* the Processor evaluating accumulated state at each step.

### The instruction set is "things Myelin can do"

The power of this approach: adding a new capability instantly makes it available to every conversation pattern.

```
Today's capabilities:
  fetch_thread, summarize_with_instruct, enrich_memories,
  web_search, embed_and_match, synthesize

Add bluesky_search tomorrow:
  → every existing task graph pattern can now include bluesky searches
  → no language changes, no interpreter updates, no new syntax
  → it's just a new event kind that flows through the same conversation

Add a code execution sandbox next month:
  → tool_chain conversations can now include compute steps
  → the Processor evaluates "should I run this code?" the same way
    it evaluates "should I fetch this URL?"
  → same conversation, same tracking, same observability
```

The conversation doesn't care what the steps *are*. It tracks them as events and lets the Processor reason about the accumulated state. New tools = new vocabulary, same grammar.

### When you DO need an embedded interpreter

Conversations-as-composition handle *orchestration* — deciding what to do, in what order, with what dependencies. But some tasks require *computation*:

- Data transformation (parse CSV, restructure JSON, extract fields)
- Numerical analysis (aggregate metrics, compute percentiles, trend detection)
- Structured format parsing (XML, protocol buffers, binary formats)
- Text manipulation beyond what a model should do (regex, templating)

For these, the right answer is an embedded interpreter (Lua, WASM, or a sandboxed Elixir evaluation). This would appear as another tool in the conversation — a `:compute` event kind — and would benefit from the same conversation tracking, observability, and error handling as every other step.

The key distinction: **orchestration is conversations, computation is interpreters.** Don't use a model to do math. Don't use a scripting engine to decide what to research. Each mechanism for its strengths.

### Emergent composition

Because the Processor can evaluate conversation state and decide next steps, novel workflows emerge at runtime without being pre-programmed:

```elixir
# Processor sees: web_search returned 12 results about "atproto federation"
# Processor decides: 3 look relevant, fetch those specifically
# No one programmed "if search returns > 10 results, filter and deep-fetch"
# The Processor made a judgment call based on conversation state

# Later: one of the fetched pages mentions a person in the entity registry
# Processor decides: search engram for prior interactions with this person
# Emergent connection — the conversation branched based on content, not rules
```

This is the qualitative difference between a task runner and an agent. A task runner executes a pre-defined graph. An agent evaluates accumulated state and decides what to do next. Conversations give the agent a *memory of what it's already tried*, which is the prerequisite for intelligent next-step selection.

---

## Delegated Execution

The conversation architecture enables a powerful pattern: a higher-tier model *tasks* a lower-tier model, which carries out the work in its own conversation, then reports back. The higher tier evaluates the results without ever touching the raw work.

### The pattern

```
Judgment tier (Sonnet):
  "collect deep system metrics from srv-001.local,
   if there are disk issues get a full SMART report"
       │
       ▼  creates task conversation, delegates to worker
Worker tier (32B / finetune):
       │  opens SSH conversation with srv-001
       │  runs df, vmstat, iostat, checks dmesg
       │  sees disk errors → decides to run smartctl (branching!)
       │  parses output, extracts findings
       │  closes SSH conversation
       ▼  reports back in task conversation
Judgment tier (Sonnet):
  evaluates report → "disk 2 showing reallocated sectors,
  trending toward failure, recommend replacement within 30 days"
```

Three conversations are involved:

| Conversation | Source | Participants | Purpose |
|---|---|---|---|
| Task conversation | `:tool_chain` | Sonnet ↔ Worker | High-level job: "diagnose srv-001" |
| SSH conversation | `:bash` | Worker ↔ srv-001 | Terminal session: the actual commands |
| Parent conversation | varies | Whatever triggered the task | Original context (alert, user request, etc.) |

### Isolation by design

The critical property: **Sonnet never sees raw terminal output.** It sees the worker's report — already extracted, structured, relevant. The SSH conversation might contain 200 lines of `iostat` output, but what flows up to the task conversation is "disk 2: 98% util, avg queue 47, SMART: 832 reallocated sectors, trend: +12/month."

This is context management through architecture, not prompt engineering. Each conversation is a scope boundary. Raw data stays in the worker's conversation. Processed findings flow up to the task conversation. Judgment flows up to the parent conversation.

```elixir
# Task conversation events (what Sonnet sees):
[
  %Event{kind: :task_delegated, summary: "diagnose srv-001, check disks"},
  %Event{kind: :task_result, summary: """
    srv-001 diagnostics complete. 4 disks checked.
    disk 2 (sdb): degraded — 832 reallocated sectors (+12/month),
    98% utilization, queue depth 47. SMART overall: PASSED but trending
    toward failure. Other disks healthy.
    Full SMART report attached for disk 2.
    """}
]

# SSH conversation events (what the worker generated — Sonnet never sees these):
[
  %Event{kind: :tool_call, summary: "ssh srv-001.local"},
  %Event{kind: :tool_result, summary: "connected"},
  %Event{kind: :tool_call, summary: "df -h"},
  %Event{kind: :tool_result, summary: "... 200 lines ..."},
  %Event{kind: :tool_call, summary: "smartctl -a /dev/sdb"},
  %Event{kind: :tool_result, summary: "... 150 lines ..."},
  # ... etc
]
```

### Worker intelligence vs. scripting

The worker model (32B, or a task-specific finetune) is *not* running a script. It received an instruction — "collect deep metrics, check for disk issues" — and exercises judgment about how to carry it out. It decides which commands to run, in what order, and branches based on what it finds ("disk errors in dmesg → run smartctl"). This is qualitatively different from a shell script because:

- The instruction is natural language, not a procedure
- The worker adapts to unexpected output (new error types, unusual configurations)
- Branching decisions are based on *understanding*, not pattern matching
- The worker can ask for clarification by reporting back to the task conversation

But the worker is also cheap enough to throw at a 15-minute terminal session without budget anxiety. The 32B model running locally costs compute time, not API dollars. This is where the hardware scaling story matters — see VISION-beyond-social.md.

### Conversation type routing to specialized workers

The conversation's `conversation_type` classification — which the Processor already computes for compaction policy — doubles as a worker routing signal:

| Conversation type | Preferred worker | Why |
|---|---|---|
| `:maintenance` | Sysadmin finetune | Terminal commands, log parsing, system diagnostics |
| `:research_session` | Information extraction finetune | Web content, document parsing, citation tracking |
| `:technical_discussion` | Code/architecture finetune | Code review, refactoring, architecture analysis |
| `:support` | General instruct model | Flexible, customer-facing, needs broad knowledge |
| `:administrative` | General instruct model | Scheduling, coordination, email drafting |

One classification, three downstream uses: compaction policy, dashboard display, and worker routing. The Processor's 5-token classification call keeps paying dividends.

### Nested delegation

Nothing prevents a worker from delegating further. A 32B model tasked with "audit the security posture of srv-001" might:
1. Open an SSH conversation to collect data
2. Open a `:tool_chain` conversation to search CVE databases for the installed package versions
3. Synthesize both into a report

Each sub-conversation tracks independently. Each gets its own compaction policy. The 32B's task conversation accumulates the synthesized results. Sonnet sees only the final report.

This is recursive — conversations within conversations, each tier doing what it's best at, with the conversation architecture providing tracking, isolation, and observability at every level.

---

## Output Conversation Tagging

Outputs need conversation tagging too — when Myelin replies to a thread, the reply belongs to the same conversation as the events that triggered it. The model generating the output should NOT have to think about this. It's automatic.

### Schema additions

**Event schema** — already has `signals.thread_id` which serves as the conversation identifier at ingestion. No change needed, but the enrichment step now injects `signals.conversation` (the `%ConversationContext{}` struct). Documented in the Integration with Event Enrichment section above.

**OutputEvent schema** — gains a `conversation_id` field:

```elixir
defmodule Myelin.OutputEvent do
  @type t :: %__MODULE__{
    # ... existing fields ...

    # conversation context (set automatically by OutputPipeline)
    conversation_id: String.t() | nil,    # which conversation this output belongs to

    # ... rest of schema ...
  }
end
```

### Automatic tagging pipeline

The model doesn't tag outputs with conversation IDs. The OutputPipeline does, deterministically:

```elixir
defmodule Myelin.OutputPipeline do
  defp tag_conversation(%OutputEvent{} = output) do
    conversation_id = cond do
      # 1. Replying to an event? Use that event's conversation.
      output.source_event_id != nil ->
        case EventStore.get(output.source_event_id) do
          %Event{signals: %{thread_id: tid}} when tid != nil -> tid
          _ -> nil
        end

      # 2. Has an explicit thread target? That IS the conversation.
      output.target.thread_id != nil ->
        output.target.thread_id

      # 3. Part of an active session? Use the session's primary conversation.
      output.source_session_id != nil ->
        case SessionStore.get(output.source_session_id) do
          %Session{primary_thread_id: tid} -> tid
          _ -> nil
        end

      # 4. No conversation context — this is a new thread / unprompted output.
      true -> nil
    end

    %{output | conversation_id: conversation_id}
  end
end
```

The logic is simple: **trace back to the conversation that caused this output.** The model says "reply to this event" or "post in this thread" — the pipeline resolves the conversation ID from that.

### Post-delivery feedback

When an output is delivered, the delivery receipt feeds back into the conversation:

```elixir
defp handle_delivery_receipt(%OutputEvent{conversation_id: conv_id} = output, receipt)
     when conv_id != nil do
  # Update conversation: we participated
  ConversationRegistry.touch(conv_id, %Event{
    source: :internal,
    kind: :agent_output,
    sender: "myelin",
    signals: %{thread_id: conv_id},
    summary: "Agent #{output.action}: #{truncate(output.content.text, 80)}"
  })

  # Promote tracking level if we just participated for the first time
  conv = ConversationRegistry.lookup(conv_id)
  if conv && conv.our_turn_count == 0 do
    ConversationRegistry.set_tracking_level(conv_id, :active)
  end
end
```

This means: when Myelin replies to a thread, the conversation automatically gets updated — `our_turn_count` increments, `agent_role` becomes `:participant`, and if this is our first reply, tracking level promotes to `:active`. The model didn't have to think about any of this.

---

## Processor Operations

New and modified Processor operations for conversation support:

```elixir
# NEW: Summarize a conversation (replaces ad-hoc thread compression)
:summarize_conversation
# Input: %Conversation{} with recent events
# Output: narrative summary, topic keywords, sentiment
# Used by: ConversationRegistry maintenance, session compaction, context assembly

# NEW: Classify conversation type
:classify_conversation
# Input: %Conversation{} with participants + recent content
# Output: conversation_type atom + confidence
# Used by: ConversationRegistry (periodic reclassification)

# NEW: Extract conversation state for tool sessions
:extract_conversation_state
# Input: %Conversation{} where source == :tool_chain
# Output: structured summary of what's been tried, what worked, what's pending
# Used by: TaskScheduler when resuming interrupted tool chains

# MODIFIED: summarize_memories
# Now accepts optional conversation_id to scope memory search
:summarize_memories
# Additional arg: conversation_id (filters engram hits to conversation-relevant memories)

# MODIFIED: draft_response
# Now receives ConversationContext to maintain conversational coherence
:draft_response
# Additional context: recent conversation summary, participant dynamics
```

---

## Cache Integration

Conversation context goes into the **hot zone**, not the frozen zone — it changes per-event:

```
HOT ZONE (when event is from a known conversation):
┌──────────────────────────────────────────────┐
│ CURRENT EVENT                                │
│ [event summary + signals]                    │
│                                              │
│ CONVERSATION CONTEXT                   ← NEW │
│ Thread: at://did:plc:xyz/post/abc123         │
│ Participants: umbra.blue (tier-1, 8 turns),  │
│   otherperson.bsky (tier-2, 3 turns)         │
│ Type: technical | Trend: rising              │
│ Summary: "Discussion about discontinuous     │
│   identity and how memory systems handle..." │
│ Our role: participant (last: 2h ago)         │
│                                              │
│ PROCESSOR RESULTS                            │
│ [memory briefing, entity context, etc.]      │
└──────────────────────────────────────────────┘
```

Token budget: ~200-400 tokens for conversation context in the hot zone. The summary is the expensive part — keep it under 200 tokens (Processor's job during `summarize_conversation`).

---

## Storage

```sql
CREATE TABLE conversations (
  id TEXT PRIMARY KEY,                    -- normalized platform conversation_id
  source TEXT NOT NULL,
  state TEXT NOT NULL DEFAULT 'open',
  tracking_level TEXT NOT NULL DEFAULT 'untracked',

  -- participants (JSON array)
  participants TEXT,

  -- topology
  root_event_id TEXT,
  branch_count INTEGER NOT NULL DEFAULT 0,
  depth INTEGER NOT NULL DEFAULT 0,
  turn_count INTEGER NOT NULL DEFAULT 0,
  our_turn_count INTEGER NOT NULL DEFAULT 0,

  -- classification
  topic_signature BLOB,                   -- embedding vector (F32 BLOB for sqlite-vec)
  topic_keywords TEXT,                    -- JSON array
  conversation_type TEXT,
  sentiment TEXT,

  -- salience
  cumulative_salience REAL NOT NULL DEFAULT 0.0,
  peak_salience REAL NOT NULL DEFAULT 0.0,
  salience_trend TEXT NOT NULL DEFAULT 'stable',
  last_salience REAL NOT NULL DEFAULT 0.0,

  -- agent relationship
  policy_overrides TEXT,                  -- JSON object
  last_agent_action TEXT,                 -- ISO datetime
  agent_role TEXT,

  -- lifecycle
  opened_at TEXT NOT NULL,
  last_activity TEXT NOT NULL,
  idle_since TEXT,
  closed_at TEXT,

  -- summary
  summary TEXT,
  summary_updated_at TEXT,
  summary_covers_turn INTEGER NOT NULL DEFAULT 0
);

-- Hot path: event enrichment lookup
CREATE INDEX idx_conversations_id ON conversations(id);

-- Management: find active conversations
CREATE INDEX idx_conversations_tracking ON conversations(tracking_level)
  WHERE tracking_level IN ('watched', 'active', 'interactive');

-- Maintenance: find idle conversations for decay
CREATE INDEX idx_conversations_state_activity ON conversations(state, last_activity)
  WHERE state = 'open';

-- Entity queries: conversations involving a specific person
-- (requires a junction table or JSON query — defer to implementation)
```

---

## Supervision Tree Addition

```
Interface.Supervisor (:one_for_one)
├── EventPipeline
├── OutputPipeline
├── Interface.Registry
├── ConversationRegistry          ← NEW (after EventPipeline, before interfaces)
├── Interface.Timer
├── Interface.Bluesky
└── Interface.Telegram
```

ConversationRegistry starts after EventPipeline (it needs PubSub) and before interfaces (they'll start emitting events that need conversation lookups).

---

## Maintenance

ConversationRegistry runs periodic maintenance (via TaskScheduler or internal timer):

**Every 60s:**
- Decay idle check: conversations with no activity past threshold → demote tracking level
- Salience trend recalculation for `:watched` and above

**Every 10min:**
- Summary refresh: for `:active` conversations, schedule Processor `summarize_conversation` if summary is stale (covers_turn < turn_count - 5)

**Every 1h:**
- Classification refresh: Processor `classify_conversation` for conversations with significant new activity
- Close untracked conversations with no activity for 24h and turn_count < 3
- Archive closed conversations older than 7 days

**Every 6h (via Processor speculative schedule):**
- Deep analysis: topic keyword extraction, participant dynamics, sentiment trends for `:watched`+ conversations

---

## Conversation Type Vocabulary

Classified by the Processor (`classify_conversation` — a ~5-token classification call that shapes all downstream behavior). The type determines summarization strategy, compaction policy, and how the conversation is presented in context assembly.

| Type | Description | Summarization strategy |
|---|---|---|
| `:technical_discussion` | Architecture, code, systems, problem-solving | Extract decisions, conclusions, open questions |
| `:casual` | General social chat, catching up | Light summary, mostly drop — preserve only notable facts |
| `:shitposting` | Low information density, high entertainment value | Minimal retention — note participants and vibe, drop content |
| `:creative` | Poetry, fiction, art, collaborative writing | Preserve *form*, not just content — summaries must retain structure |
| `:debate` | Multiple positions, argumentation | Track who-said-what, extract positions and counterarguments |
| `:support` | Someone asking for help | Extract the problem and the resolution (or lack thereof) |
| `:announcement` | One-to-many broadcast, low interaction | Preserve the announcement, drop reactions unless notable |
| `:research_session` | Tool chain conversations | Extract methodology, findings, dead ends, and final result |
| `:administrative` | Scheduling, coordination, logistics | Extract action items and commitments with dates |
| `:maintenance` | Myelin-internal tasks (cache, rules, compaction) | Extract what was done, what changed, any anomalies |

The `:maintenance` type is particularly important — it means Myelin's own internal work (rule pruning, cache rotation, scheduled compaction) flows through the same conversation tracking and gets the same observability. "What did the system do while I was away?" becomes answerable for both social activity and self-maintenance.

Types are not fixed at classification time. A `:casual` conversation can evolve into `:technical_discussion` — the Processor reclassifies on the hourly maintenance cycle when conversation content has shifted significantly.

---

## Compaction Policy

When a conversation closes or archives, the system decides *how much to remember* about it. This is not a rule — it's a function of multiple signals, producing one of four compaction tiers.

### Compaction tiers

| Tier | Action | Cost |
|---|---|---|
| `:drop` | No engram write. Conversation record pruned. | Zero |
| `:oneliner` | Single-sentence engram note. Conversation record pruned. | ~10 Processor tokens |
| `:summary` | Full Processor summary → engram write. Raw events prunable after. | ~200 Processor tokens |
| `:deep` | Processor summary reviewed by Sonnet (if cache window available). Multiple engram memories. | ~200 Processor + opportunistic Sonnet |

### Compaction function

```elixir
defmodule Myelin.Conversation.Compaction do
  @moduledoc """
  Determines how thoroughly a conversation should be remembered.
  Pure function — no side effects, no inference calls.
  """

  def compaction_tier(%Conversation{} = conv, opts \\ []) do
    novelty = Keyword.get(opts, :novelty_score, 0.0)  # from engram overlap check

    score = 0.0
      |> add_salience_score(conv)
      |> add_tracking_score(conv)
      |> add_participant_score(conv)
      |> add_type_score(conv)
      |> add_depth_score(conv)
      |> add_novelty_score(novelty)

    cond do
      score < 0.2 -> :drop
      score < 0.4 -> :oneliner
      score < 0.7 -> :summary
      true -> :deep
    end
  end

  # Salience: was this conversation actually interesting?
  defp add_salience_score(score, %{cumulative_salience: s, peak_salience: p}) do
    score + min(s * 0.3, 0.3) + if(p > 0.8, do: 0.1, else: 0.0)
  end

  # Tracking: how much attention did we give it?
  defp add_tracking_score(score, %{tracking_level: level}) do
    score + case level do
      :untracked -> 0.0
      :watched -> 0.1
      :active -> 0.2
      :interactive -> 0.3
    end
  end

  # Participants: anyone we care about?
  defp add_participant_score(score, %{participants: participants}) do
    has_tier1 = Enum.any?(participants, &(&1[:tier] == 1))
    has_agent = Enum.any?(participants, &(&1[:role] == :agent))
    score + if(has_tier1, do: 0.15, else: 0.0) + if(has_agent, do: 0.1, else: 0.0)
  end

  # Type: some conversation types are inherently more worth remembering
  defp add_type_score(score, %{conversation_type: type}) do
    score + case type do
      :technical_discussion -> 0.15
      :debate -> 0.1
      :support -> 0.1       # resolutions are valuable
      :research_session -> 0.15
      :creative -> 0.1      # preserve art
      :maintenance -> 0.05  # log anomalies, drop routine
      :casual -> 0.0
      :shitposting -> -0.1  # actively de-prioritize
      :announcement -> 0.0
      :administrative -> 0.05
      _ -> 0.0
    end
  end

  # Depth: longer conversations have more potential signal
  defp add_depth_score(score, %{turn_count: turns}) do
    score + min(turns / 50.0, 0.15)  # caps at 50 turns → 0.15
  end

  # Novelty: does this add NEW knowledge vs what's already in engram?
  defp add_novelty_score(score, novelty) when is_float(novelty) do
    score + novelty * 0.25  # novelty 0.0-1.0 → 0.0-0.25 contribution
  end
end
```

### Novelty scoring

The novelty component deserves special attention. Before compaction, the system checks the conversation's topic signature against existing engram memories:

```elixir
defp compute_novelty(%Conversation{topic_signature: sig}) when is_list(sig) do
  existing = Engram.search_by_embedding(sig, top_k: 5)

  case existing do
    [] -> 1.0                          # totally new topic — maximum novelty
    hits ->
      max_similarity = hits |> Enum.map(& &1.similarity) |> Enum.max()
      1.0 - max_similarity             # high overlap = low novelty
  end
end
```

**Salience gets you noticed, novelty gets you remembered.** A high-salience conversation that covers ground already well-represented in engram gets a `:oneliner` at best. A moderate-salience conversation introducing genuinely new information gets a full `:summary`. This prevents memory bloat from repetitive discussions while ensuring novel insights are captured even from quieter threads.

---

## Design Properties

- **Conversations are passive context, not controllers.** Like sessions, they don't drive behavior — the router and state machine do. Conversations provide accumulated context that makes routing decisions better.

- **The hot path is microseconds.** `ConversationRegistry.lookup/1` is an ETS read. `touch/2` is an ETS write + async persist. Event enrichment adds one parallel task that costs microseconds. Zero impact on the routing fast path.

- **Tracking level is the conversation's attention budget.** `:untracked` = free to ignore. `:watched` = cheap passive monitoring. `:active` = investing Processor cycles in summaries. `:interactive` = full inference resources allocated. The level controls how much the system is willing to spend on this conversation, exactly mirroring the state machine's attention economy at a per-conversation granularity.

- **Platform-agnostic by construction.** A Bluesky thread and a Telegram chat are the same type with different `source` values. The interface stamps the natural ID, the inside does the rest. Adding a new platform means adding a new interface — the conversation system requires zero changes.

- **Crash-safe by design.** Interface crashes lose nothing. ConversationRegistry rebuilds from SQLite on restart. Events that arrive during the brief restart window queue in EventPipeline's bounded buffer and are processed normally once the registry is back.

- **Tool sessions and social conversations are the same thing.** A research session is a conversation with `source: :tool_chain`. A Bluesky thread is a conversation with `source: :bluesky`. The same operations (summarize, classify, extract state) work on both. This unification is the core insight — it means every investment in conversation intelligence pays off across all interaction modalities.

- **Conversations compound salience.** A thread that's been interesting for an hour doesn't need to re-prove itself on every reply. Cumulative salience and trend provide inertia — the router respects the history of the exchange, not just the instant.
