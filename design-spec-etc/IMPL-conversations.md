# Implementation Plan — Conversations

## Overview

Conversations are a first-class tracking abstraction layered on top of `thread_id`. Interfaces already stamp events with the right identifier — we just start *tracking* it. ConversationRegistry is a GenServer that owns ETS (hot-path lookups) and delegates SQLite persistence to MemoryStore via a new `Conversations` submodule.

**Key decision:** `conversation_id == thread_id`. No new field on Event. The conversation is the *record* (accumulated state about that thread_id), not a new identifier.

**Key decision:** ConversationRegistry owns tracking logic + ETS. MemoryStore owns SQLite via `MemoryStore.Conversations` submodule. When a conversation dies, ConversationRegistry hands it to MemoryStore for archival. When a thread_id shows up that might be resurrected, ConversationRegistry checks MemoryStore.

**Key decision:** All long-term storage (briefings, engram writes, rules, etc.) gains an optional `thread_id` column. "Is this a resurrected thread?" becomes a single indexed lookup.

---

## Build Order

```
Phase C1: Schema + ConversationRegistry + MemoryStore.Conversations ✅
  C1a: Conversation struct + ConversationContext struct ✅
  C1b: MemoryStore.Conversations submodule (init_db, CRUD) ✅
  C1c: ConversationRegistry GenServer (ETS + SQLite delegation) ✅
  C1d: Add thread_id column to briefings table ✅

Phase C2: Integration (parallel, all independent) ✅
  C2a: EventEnricher — conversation lookup as parallel task ✅
  C2b: Salience — conversation_component/1 ✅
  C2c: InferenceRouter — rule matching + context injection ✅
  C2d: OutputEvent — conversation_id field + auto-tagging ✅

Phase C3: Lifecycle ✅
  C3a: Tracking level transitions + decay timers ✅
  C3b: Processor recipe: :summarize_conversation (4-hour opportunistic) ✅
  C3c: Back-pressure: don't drop events from tracked conversations ✅

Phase C4: Compaction ✅
  C4a: Compaction tier scoring function ✅
  C4b: Novelty scoring against Engram ✅
  C4c: Archive pipeline ✅
```

---

## Phase C1a: Structs

### File: `lib/agent_runtime/conversation.ex` (NEW)

```elixir
defmodule AgentRuntime.Conversation do
  @moduledoc """
  A conversation is any stateful multi-turn exchange.
  Bluesky thread, Telegram chat, tool chain, sigma alert cluster — same type.
  """

  defstruct [
    :id,                       # == thread_id from the originating interface
    :source,                   # :bluesky | :telegram | :syslog | :internal | :tool_chain
    :root_event_id,            # first event we saw in this conversation

    # participants
    participants: [],           # [%{handle: String.t(), role: atom(), first_seen: DateTime.t(), last_seen: DateTime.t(), message_count: integer()}]

    # topology
    turn_count: 0,
    our_turn_count: 0,
    depth: 0,                  # max reply depth (tree-structured threads)
    branch_count: 0,

    # classification (evolves over time)
    topic_signature: nil,      # embedding vector | nil
    topic_keywords: [],
    conversation_type: nil,    # :technical | :casual | :shitposting | :research_session | :maintenance | etc.
    sentiment: nil,            # :positive | :neutral | :negative | :heated

    # salience history
    cumulative_salience: 0.0,
    peak_salience: 0.0,
    salience_trend: :stable,   # :rising | :stable | :declining | :dead
    last_salience: 0.0,

    # agent relationship
    tracking_level: :untracked,  # :untracked | :watched | :active | :interactive
    policy_overrides: %{},
    last_agent_action: nil,
    agent_role: nil,             # :observer | :participant | :driver

    # lifecycle
    state: :open,              # :open | :idle | :closed | :archived
    opened_at: nil,
    last_activity: nil,
    idle_since: nil,
    closed_at: nil,

    # summary
    summary: nil,
    summary_updated_at: nil,
    summary_covers_turn: 0
  ]

  @type t :: %__MODULE__{}
end
```

### File: `lib/agent_runtime/conversation_context.ex` (NEW)

The lightweight struct injected into event signals during enrichment. ~200 bytes in ETS.

```elixir
defmodule AgentRuntime.ConversationContext do
  @moduledoc """
  Lightweight conversation data for the enrichment/routing hot path.
  Sits in ETS alongside the full Conversation. Injected into event.signals.
  """

  defstruct [
    :id,
    :tracking_level,
    :cumulative_salience,
    :salience_trend,
    :conversation_type,
    :turn_count,
    :our_turn_count,
    :last_activity,
    policy_overrides: %{},
    topic_keywords: []
  ]

  @type t :: %__MODULE__{}

  @doc "Extract from a full Conversation struct"
  def from_conversation(%AgentRuntime.Conversation{} = conv) do
    %__MODULE__{
      id: conv.id,
      tracking_level: conv.tracking_level,
      cumulative_salience: conv.cumulative_salience,
      salience_trend: conv.salience_trend,
      conversation_type: conv.conversation_type,
      turn_count: conv.turn_count,
      our_turn_count: conv.our_turn_count,
      last_activity: conv.last_activity,
      policy_overrides: conv.policy_overrides,
      topic_keywords: conv.topic_keywords
    }
  end
end
```

---

## Phase C1b: MemoryStore.Conversations

### File: `lib/agent_runtime/memory_store/conversations.ex` (NEW)

Follows the exact pattern of `MemoryStore.Briefings`, `MemoryStore.EntityRegistry`, etc. Pure functions that take a `conn` and operate on SQLite.

```elixir
defmodule AgentRuntime.MemoryStore.Conversations do
  @moduledoc """
  SQLite persistence for conversations.
  Called by MemoryStore GenServer — never directly by other modules.
  """

  def init_db(conn) do
    Exqlite.Sqlite3.execute(conn, """
    CREATE TABLE IF NOT EXISTS conversations (
      id TEXT PRIMARY KEY,
      source TEXT NOT NULL,
      state TEXT NOT NULL DEFAULT 'open',
      tracking_level TEXT NOT NULL DEFAULT 'untracked',
      root_event_id TEXT,
      participants TEXT,
      branch_count INTEGER NOT NULL DEFAULT 0,
      depth INTEGER NOT NULL DEFAULT 0,
      turn_count INTEGER NOT NULL DEFAULT 0,
      our_turn_count INTEGER NOT NULL DEFAULT 0,
      topic_signature BLOB,
      topic_keywords TEXT,
      conversation_type TEXT,
      sentiment TEXT,
      cumulative_salience REAL NOT NULL DEFAULT 0.0,
      peak_salience REAL NOT NULL DEFAULT 0.0,
      salience_trend TEXT NOT NULL DEFAULT 'stable',
      last_salience REAL NOT NULL DEFAULT 0.0,
      policy_overrides TEXT,
      last_agent_action TEXT,
      agent_role TEXT,
      opened_at TEXT NOT NULL,
      last_activity TEXT NOT NULL,
      idle_since TEXT,
      closed_at TEXT,
      summary TEXT,
      summary_updated_at TEXT,
      summary_covers_turn INTEGER NOT NULL DEFAULT 0
    )
    """)

    # Index for the hot-path resurrection check
    Exqlite.Sqlite3.execute(conn, """
    CREATE INDEX IF NOT EXISTS idx_conversations_state_activity
    ON conversations(state, last_activity) WHERE state = 'open'
    """)

    # Index for active conversation queries
    Exqlite.Sqlite3.execute(conn, """
    CREATE INDEX IF NOT EXISTS idx_conversations_tracking
    ON conversations(tracking_level)
    WHERE tracking_level IN ('watched', 'active', 'interactive')
    """)
  end

  # CRUD functions: upsert/2, lookup/2, list_active/1, close/2, archive/2
  # See Briefings module for the pattern — prepare statement, bind, step.
  # JSON-encode participants, topic_keywords, policy_overrides via Jason.
end
```

### Changes to `lib/agent_runtime/memory_store.ex`

```
# In init/1, add:
alias AgentRuntime.MemoryStore.Conversations
Conversations.init_db(conn)

# New public API functions:
def upsert_conversation(server \\ __MODULE__, conversation)
def lookup_conversation(server \\ __MODULE__, id)
def list_active_conversations(server \\ __MODULE__)
def close_conversation(server \\ __MODULE__, id, reason)
def archive_conversation(server \\ __MODULE__, id)
def search_by_thread_id(server \\ __MODULE__, thread_id)
  # For resurrection check — single indexed lookup

# New handle_call clauses routing to Conversations submodule.
```

### Changes to `lib/agent_runtime/memory_store/briefings.ex`

Add optional `thread_id` column:

```sql
ALTER TABLE briefings ADD COLUMN thread_id TEXT;
CREATE INDEX IF NOT EXISTS idx_briefings_thread_id ON briefings(thread_id);
```

Use `ALTER TABLE IF NOT EXISTS` pattern or guard with a column-existence check, since this is a migration on existing data.

---

## Phase C1c: ConversationRegistry

### File: `lib/agent_runtime/conversation_registry.ex` (NEW)

GenServer owning ETS for hot-path reads. Delegates persistence to MemoryStore.

```elixir
defmodule AgentRuntime.ConversationRegistry do
  use GenServer

  # --- Hot path (called during enrichment, must be microseconds) ---

  def lookup(server \\ __MODULE__, thread_id)
    # ETS read → returns %ConversationContext{} | nil

  def touch(server \\ __MODULE__, thread_id, %Event{})
    # ETS write: update last_activity, increment turn_count,
    # update participant last_seen.
    # Async: GenServer.cast for SQLite persist.

  # --- Warm path ---

  def get_or_create(server \\ __MODULE__, thread_id, %Event{})
    # If not in ETS: check MemoryStore (resurrection).
    # If not in MemoryStore: create new conversation.
    # Returns %ConversationContext{}.

  def set_tracking_level(server \\ __MODULE__, thread_id, level, opts \\ [])
  def update_summary(server \\ __MODULE__, thread_id, summary, covers_turn)
  def active_conversations(server \\ __MODULE__)

  # --- Cold path ---

  def close(server \\ __MODULE__, thread_id, reason)
    # Remove from ETS, persist final state to MemoryStore.

  def request_tag(server \\ __MODULE__, source, metadata)
    # Generate a new conversation_id for tool chains / research threads.
    # Returns a ULID. The caller stamps it as thread_id on events.
    # This is how Engaged/Processor request a tag for a new research thread.

  # --- GenServer internals ---

  # State: %{ets_table: ref, memory_store: pid/name}
  # ETS stores: {thread_id, %Conversation{}} and {thread_id, :context, %ConversationContext{}}
  # Maintenance timer: every 60s, decay check.
  # Persistence timer: every 30s, batch-persist dirty conversations to SQLite.

  # IMPORTANT: accept :name option for test isolation (per CLAUDE.md).
end
```

### Supervision

ConversationRegistry goes into `Interface.Supervisor` after EventPipeline and before interfaces (same position as the spec suggests). It needs MemoryStore to be up (for resurrection checks on startup), and interfaces need it to be up (events need conversation lookups).

**Do NOT modify application.ex** — per CLAUDE.md, document which module should be added and the tech lead wires it post-merge.

---

## Phase C1d: thread_id on Briefings

Small migration: add `thread_id TEXT` column to briefings table. Index it. When briefings are created with a thread_id, it enables "is this a resurrected thread?" lookups.

Same pattern should eventually apply to rules table and engram writes, but briefings is the most immediately useful (search results scoped to a conversation).

---

## Phase C2a: EventEnricher Integration

### File: `lib/agent_runtime/ingester/event_enricher.ex` (MODIFY)

Add one more parallel task to `enrich/2`:

```elixir
def enrich(%Event{} = event, opts \\ []) do
  memory_store = opts[:memory_store] || MemoryStore
  conversation_registry = opts[:conversation_registry] || ConversationRegistry

  if alive?(memory_store) do
    # Existing parallel tasks (unchanged)
    entity_task = Task.async(fn -> MemoryStore.lookup_entity(memory_store, event.sender) end)
    text = (event.payload && event.payload["text"]) || ""
    keyword_task = Task.async(fn -> MemoryStore.match_keywords(memory_store, text) end)
    hot_topic_task = Task.async(fn -> MemoryStore.check_hot_topics(memory_store, text) end)

    # NEW: conversation lookup (also parallel, also microseconds — ETS read)
    conversation_task = Task.async(fn ->
      case event.thread_id do
        nil -> nil
        thread_id ->
          case ConversationRegistry.lookup(conversation_registry, thread_id) do
            nil ->
              # Unknown thread — create a new conversation record
              ConversationRegistry.get_or_create(conversation_registry, thread_id, event)
            ctx ->
              # Known thread — touch it (update last_activity, increment turn_count)
              ConversationRegistry.touch(conversation_registry, thread_id, event)
              ctx
          end
      end
    end)

    entity = Task.await(entity_task)
    keyword_hits = Task.await(keyword_task)
    hot_topic_score = Task.await(hot_topic_task)
    conversation_ctx = Task.await(conversation_task)

    signals = %{
      sender_tier: entity && entity.tier,
      entity_salience: (entity && entity.salience_score) || 0.0,
      keyword_hits: keyword_hits || [],
      hot_topic_score: hot_topic_score || 0.0,
      conversation: conversation_ctx  # NEW — %ConversationContext{} | nil
    }

    %{event | signals: Map.merge(event.signals || %{}, signals)}
  else
    %{event | signals: event.signals || %{}}
  end
end
```

### Test changes

All existing EventEnricher tests pass unchanged (conversation field is additive). New tests:
- Event with thread_id → signals.conversation is populated
- Event without thread_id → signals.conversation is nil
- Unknown thread_id → new conversation created
- Known thread_id → touch updates turn_count

---

## Phase C2b: Salience — Conversation Component

### File: `lib/agent_runtime/salience.ex` (MODIFY)

Add a `conversation_component/1` to the score calculation:

```elixir
def score(%Event{} = event, signals \\ %{}) do
  signals = Map.merge(event.signals || %{}, signals)

  # ... existing components unchanged ...

  # NEW: Conversation component (capped at 0.4)
  conversation_score = conversation_component(signals[:conversation])

  base = sender_tier_score + hot_topic_score + entity_salience + keyword_contribution + conversation_score

  # ... existing multiplier logic unchanged ...
end

defp conversation_component(nil), do: 0.0
defp conversation_component(%ConversationContext{} = ctx) do
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

  participation_boost = if ctx.our_turn_count > 0, do: 0.1, else: 0.0

  min(tracking_boost + trend_boost + participation_boost, 0.4)
end
```

### Test changes

Existing salience tests pass unchanged (conversation defaults to nil → 0.0 contribution). New tests:
- Event in :watched conversation → +0.1 boost
- Event in :active rising conversation → +0.3 boost
- Event in :untracked declining conversation → -0.05

---

## Phase C2c: InferenceRouter — Rule Matching + Context

### File: `lib/agent_runtime/inference_router.ex` (MODIFY)

Two changes:

**1. Rule matching** — add `conversation_id` to `rule_matches?/2`:

```elixir
# In rule_matches?/2 (line ~185), add:
match_field?(target[:conversation_id], event.thread_id) and
```

This means rules can target specific conversations (boost_thread with a conversation_id target already works conceptually — this just makes the matching explicit).

**2. Speculative prep** — use conversation context for smarter prep:

```elixir
# In do_route_event, where speculative prep fires (line ~73):
# Change context_id from event.thread_id to include conversation awareness.
# If conversation is :active or :watched, prep is more valuable — raise priority.

prep_priority = case event.signals[:conversation] do
  %{tracking_level: level} when level in [:active, :interactive] -> :real
  _ -> :speculative
end

AgentRuntime.Processor.async(:summarize_memories, %{memories: memories},
  priority: prep_priority, context_id: event.thread_id)
```

**3. Vibe prompt injection** — pass conversation context to Templates.render_vibe:

```elixir
# In perform_two_pass_flow, pass conversation to template:
conversation_ctx = event.signals[:conversation]
vibe_prompt = Templates.render_vibe(event, current_state_name, conversation: conversation_ctx)
```

This requires a small change to `Templates.render_vibe/3` to accept and format conversation context. Optional line like:
```
Conversation: tracked (watched), 12 turns, rising salience, topic: "identity/continuity"
```

### File: `lib/agent_runtime/salience/rules_applicator.ex` (MODIFY)

Add conversation_id matching (mirrors the change in InferenceRouter):

```elixir
# In rule_matches?/2 (line ~32), add:
match_conversation?(target[:conversation_id], event.thread_id) and

defp match_conversation?(nil, _), do: true
defp match_conversation?(target_id, event_thread_id), do: match_field?(target_id, event_thread_id)
```

### File: `lib/agent_runtime/memory_store/rule.ex` (MODIFY)

Add `:boost_conversation` and `:suppress_conversation` to `rule_kind` type. Add `conversation_id` to `target` type.

---

## Phase C2d: OutputEvent — Conversation Tagging

### File: `lib/agent_runtime/output_event.ex` (MODIFY)

Add `conversation_id` field:

```elixir
defstruct [
  # ... existing fields ...
  :conversation_id,  # NEW — auto-tagged by OutputPipeline
]
```

### File: `lib/agent_runtime/output_pipeline.ex` (MODIFY)

Add auto-tagging step. After validation, before delivery:

```elixir
defp tag_conversation(%OutputEvent{} = output) do
  conversation_id = cond do
    # 1. Has source_event_id? Trace back to that event's thread_id.
    output.source_event_id != nil ->
      # The source event's thread_id IS the conversation_id.
      # We don't have an event store yet, so pull from target metadata.
      output.target[:thread_id] || output.target[:chat_id]

    # 2. Has explicit thread target?
    output.target[:thread_id] != nil ->
      output.target[:thread_id]

    # 3. No conversation context.
    true -> nil
  end

  %{output | conversation_id: conversation_id}
end
```

After delivery, update ConversationRegistry:

```elixir
defp handle_delivery_success(output) do
  if output.conversation_id do
    ConversationRegistry.record_agent_action(output.conversation_id, output)
    # This increments our_turn_count and may promote tracking_level to :active
  end
end
```

---

## Phase C3a: Tracking Level Transitions + Decay

### In ConversationRegistry (maintenance timer)

Every 60 seconds, run decay check:

```
:interactive  ──(30min silence)──→  :active
:active       ──(2h silence)────→  :watched
:watched      ──(6h silence)────→  :untracked
:untracked    ──(24h silence)───→  :closed (if turn_count < 3)
                                   :idle (if turn_count >= 3)
```

Promotion rules (called from enrichment and routing):
- Router escalation on a conversation event: :untracked → :watched
- Agent participates (OutputEvent delivered): :watched → :active (or :untracked → :active)
- Human connects (:human_connect event): any → :interactive
- Rule: boost_conversation sets tracking_level directly

### Policy overrides

Conversations carry `policy_overrides` map:
```elixir
%{
  min_tracking_level: :watched,     # decay won't drop below this
  suppress_until: ~U[...],          # temporarily ignore
  bypass_router: true,              # events skip two-pass flow
  output_priority: :urgent          # outputs get :urgent priority
}
```

Set by rules (`:boost_conversation` kind), agent decisions (Sonnet tool call), or Interactive mode.

---

## Phase C3b: Processor Recipe — :summarize_conversation

### File: `lib/agent_runtime/processor/recipes.ex` (MODIFY)

Add new recipe:

```elixir
conversation_summary: [
  {:fetch_conversation_events, []},
  {:generate_summary, [max_tokens: 200]},
  {:update_conversation_summary, []}
]
```

**Trigger:** ConversationRegistry maintenance timer, every 4 hours, for conversations where:
- `tracking_level` in `[:watched, :active, :interactive]`
- `summary_covers_turn < turn_count - 5` (at least 5 new turns since last summary)

The Processor calls ConversationRegistry to get recent events, generates a summary via llama-server, and writes it back via `ConversationRegistry.update_summary/4`.

**Future optimization:** Haiku/Sonnet can write summaries as a side effect during Attentive/Engaged processing. A few extra output tokens. But that's a later iteration — start with the explicit recipe.

---

## Phase C3c: Back-Pressure Conversation Awareness

### File: `lib/agent_runtime/event_pipeline.ex` (MODIFY)

In `drop_lowest_salience/2`, protect events from tracked conversations:

```elixir
defp drop_lowest_salience(queue, new_event) do
  all_events = :queue.to_list(queue) ++ [new_event]

  # Partition: tracked conversations are protected
  {protected, droppable} = Enum.split_with(all_events, fn e ->
    case e.signals[:conversation] do
      %{tracking_level: level} when level in [:watched, :active, :interactive] -> true
      _ -> false
    end
  end)

  # Only drop from the droppable set
  if droppable == [] do
    # Everything is protected — fall back to dropping lowest salience from all
    lowest = Enum.min_by(all_events, &(&1.signals[:salience_score] || 0.0))
    # ... existing logic ...
  else
    lowest = Enum.min_by(droppable, &(&1.signals[:salience_score] || 0.0))
    # ... drop from droppable set, rebuild queue ...
  end
end
```

---

## Phase C3d: Research Thread Tags

### ConversationRegistry.request_tag/3

When Engaged or Processor decides to start a research thread:

```elixir
def request_tag(server \\ __MODULE__, source, metadata) do
  GenServer.call(server, {:request_tag, source, metadata})
end

# In handle_call:
def handle_call({:request_tag, source, metadata}, _from, state) do
  tag = Ulid.generate()
  conversation = %Conversation{
    id: tag,
    source: source,  # :tool_chain or :research
    state: :open,
    tracking_level: :active,  # research threads start tracked
    opened_at: DateTime.utc_now(),
    last_activity: DateTime.utc_now(),
    policy_overrides: %{bypass_router: true}  # tool returns don't need routing
  }
  # Insert into ETS + persist
  {:reply, tag, state}
end
```

The caller stamps this tag as `thread_id` on all events in the chain. When results come back, the router sees the conversation's `bypass_router: true` override and skips two-pass routing. Results flow back to the session that ordered the research because the conversation tracks who initiated it.

---

## Test Strategy

### New test files:
- `test/agent_runtime/conversation_test.exs` — struct creation, ConversationContext.from_conversation
- `test/agent_runtime/conversation_registry_test.exs` — GenServer: lookup, touch, get_or_create, decay, request_tag
- `test/agent_runtime/memory_store/conversations_test.exs` — SQLite CRUD

### Modified test files:
- `test/agent_runtime/event_enricher_test.exs` — verify conversation signal injection
- `test/agent_runtime/salience_test.exs` — verify conversation_component scoring
- `test/agent_runtime/inference_router_test.exs` — verify conversation_id rule matching
- `test/agent_runtime/output_pipeline_test.exs` — verify conversation tagging
- `test/agent_runtime/event_pipeline_test.exs` — verify back-pressure protection

### Test isolation:
- ConversationRegistry accepts `:name` option (per CLAUDE.md pattern)
- Tests start their own instance with unique name
- ETS table name derived from GenServer name to avoid collisions

---

## Worker Assignment Matrix

| Worker | Phase | Files to Create/Modify | Dependencies |
|--------|-------|----------------------|--------------|
| **conv-structs** | C1a | conversation.ex, conversation_context.ex | None |
| **conv-memstore** | C1b + C1d | memory_store/conversations.ex, memory_store.ex, memory_store/briefings.ex | C1a (struct definitions) |
| **conv-registry** | C1c | conversation_registry.ex | C1a + C1b (structs + persistence) |
| **conv-enricher** | C2a | ingester/event_enricher.ex | C1c (registry API) |
| **conv-salience** | C2b | salience.ex | C1a (ConversationContext struct) |
| **conv-router** | C2c | inference_router.ex, salience/rules_applicator.ex, memory_store/rule.ex, router/templates.ex | C1a (ConversationContext) |
| **conv-output** | C2d | output_event.ex, output_pipeline.ex | C1c (registry API) |
| **conv-lifecycle** | C3a + C3d | conversation_registry.ex (extend) | C1c complete |
| **conv-processor** | C3b | processor/recipes.ex | C1c + C2a |
| **conv-backpressure** | C3c | event_pipeline.ex | C2a (conversation in signals) |

### Parallelization:
- **Wave 1:** conv-structs (C1a) — no dependencies, ~30 min
- **Wave 2:** conv-memstore (C1b+C1d) + conv-salience (C2b) — both only need C1a
- **Wave 3:** conv-registry (C1c) — needs C1a + C1b
- **Wave 4:** conv-enricher (C2a) + conv-router (C2c) + conv-output (C2d) + conv-backpressure (C3c) — all independent, all need C1c
- **Wave 5:** conv-lifecycle (C3a+C3d) + conv-processor (C3b) — needs wave 4

---

## Anti-Patterns for Workers

These are the known Gemini failure modes — include in every worker spec:

1. **Do NOT modify application.ex.** Document supervisor additions in commit message.
2. **Do NOT hardcode GenServer names.** Accept `:name` option. Use `opts[:name] || __MODULE__`.
3. **Run `mix compile --warnings-as-errors && mix test` before reporting done.** Not just your tests — the FULL suite.
4. **Wrap `on_exit` cleanup in `try/catch :exit, _ -> :ok`** for GenServer.stop calls.
5. **If in a worktree, `git add -A && git commit` before reporting.** Uncommitted changes are lost on merge.
6. **Do NOT create mock infrastructure.** All local endpoints (toybox, cortex) are real and live but NOT CONNECTED in the worker environment. Use mocks/stubs in tests, not fake HTTP servers.
7. **Use `Exqlite.Sqlite3` directly**, not Ecto. Follow the pattern in `memory_store/briefings.ex` exactly.
8. **JSON encode lists/maps with `Jason.encode!/1`** before binding to SQLite TEXT columns. Decode with `Jason.decode!/1` on read.
9. **ETS table naming:** derive from GenServer name to avoid collisions in concurrent tests. Pattern: `:ets.new(:"#{name}_conversations", [:set, :protected])`.
10. **Do NOT add type annotations or docstrings to code you didn't write.** Only annotate new code.

---

## Files Reference

Workers should read these files to understand the patterns:

| File | Why |
|------|-----|
| `lib/agent_runtime/memory_store.ex` | GenServer pattern, init_db calls, ETS setup, public API delegation |
| `lib/agent_runtime/memory_store/briefings.ex` | SQLite submodule pattern (init_db, prepare/bind/step) |
| `lib/agent_runtime/memory_store/entity_registry.ex` | ETS + SQLite dual-store pattern |
| `lib/agent_runtime/memory_store/hot_topics.ex` | ETS-only hot-path pattern, decay timers |
| `lib/agent_runtime/ingester/event_enricher.ex` | Parallel Task.async enrichment pattern |
| `lib/agent_runtime/salience.ex` | Pure scoring function pattern |
| `lib/agent_runtime/event.ex` | Event struct (thread_id field) |
| `lib/agent_runtime/output_event.ex` | OutputEvent struct |
| `lib/agent_runtime/event_pipeline.ex` | Pipeline flow, back-pressure |
| `lib/agent_runtime/inference_router.ex` | Two-pass routing, rule matching, speculative prep |
| `lib/agent_runtime/salience/rules_applicator.ex` | Rule matching pattern |
| `lib/agent_runtime/memory_store/rule.ex` | Rule struct and types |
| `lib/agent_runtime/processor/recipes.ex` | Recipe definition pattern |
| `documentation/design/SPEC-conversation.md` | Full design spec |
