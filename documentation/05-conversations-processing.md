# Conversations, Processing & Scheduling

This document details the architecture and implementation of conversation management, asynchronous processing, and task scheduling within Myelin.

## 1. Conversation Struct

The `Myelin.Conversation` struct (defined in `lib/myelin/conversation.ex`) is the primary state container for all stateful multi-turn exchanges, including Bluesky threads, Telegram chats, and tool chains.

### Fields and Types
- **Identification & Source**
  - `id: String.t()` — The unique identifier, typically the `thread_id` from the originating interface.
  - `source: atom()` — Origin: `:bluesky`, `:telegram`, `:syslog`, `:internal`, or `:tool_chain`.
  - `root_event_id: String.t()` — The ID of the first event encountered in this conversation.
- **Topology**
  - `participants: [%{handle: String.t(), role: atom(), first_seen: DateTime.t(), last_seen: DateTime.t(), message_count: integer()}]` — List of all entities in the thread.
  - `turn_count: integer()` — Total number of messages.
  - `our_turn_count: integer()` — Number of messages sent by the agent.
  - `depth: integer()` — Maximum reply depth.
  - `branch_count: integer()` — Number of branches in tree-structured threads.
- **Classification & Salience**
  - `topic_signature: list(float())` — Embedding vector (or nil).
  - `topic_keywords: list(String.t())` — Extracted keywords.
  - `conversation_type: atom()` — e.g., `:technical`, `:casual`, `:shitposting`, `:research_session`.
  - `sentiment: atom()` — `:positive`, `:neutral`, `:negative`, `:heated`.
  - `cumulative_salience: float()` — Total salience score over the conversation life.
  - `peak_salience: float()` — Highest salience score encountered.
  - `salience_trend: atom()` — `:rising`, `:stable`, `:declining`, `:dead`.
  - `last_salience: float()` — Score of the most recent event.
- **Agent Relationship**
  - `tracking_level: atom()` — Current attention level (see below).
  - `policy_overrides: map()` — e.g., `%{bypass_router: true}` or `%{min_tracking_level: :active}`.
  - `last_agent_action: any()` — The last `OutputEvent` or action taken.
  - `agent_role: atom()` — `:observer`, `:participant`, or `:driver`.
- **Lifecycle**
  - `state: atom()` — `:open`, `:idle`, `:closed`, or `:archived`.
  - `opened_at`, `last_activity`, `idle_since`, `closed_at: DateTime.t()`.
- **Summary**
  - `summary: String.t()` — Generated text summary.
  - `summary_updated_at: DateTime.t()`.
  - `summary_covers_turn: integer()` — The turn count at which the summary was last generated.

### Lifecycle States
1. **`:open`**: Active conversation receiving events.
2. **`:idle`**: No activity for a moderate period (default > 24h of silence).
3. **`:closed`**: Explicitly ended or reached a terminal decay state.
4. **`:archived`**: Persisted to long-term storage (Engram/Briefings) and removed from the hot path.

---

## 2. ConversationContext Struct

`Myelin.ConversationContext` is a lightweight struct used for the enrichment and routing hot path. It is injected into `event.signals` during the pipeline.

### Fields
- `id`, `tracking_level`, `cumulative_salience`, `salience_trend`, `conversation_type`, `turn_count`, `our_turn_count`, `last_activity`.
- `policy_overrides`: Merged map of active overrides.
- `topic_keywords`: Keywords used for context matching.

### Usage
It is stored in ETS alongside the full `Conversation` struct but is significantly smaller, allowing fast lookups during event enrichment without loading the full conversation history.

---

## 3. ConversationRegistry

The `ConversationRegistry` is a GenServer that manages the active "working set" of conversations.

### Architecture
- **ETS Hot Path**: Uses a named ETS table (e.g., `Myelin.ConversationRegistry_conversations`) to store both full `%Conversation{}` structs and `%ConversationContext{}` structs (keyed as `{:context, thread_id}`).
- **SQLite Persistence**: Delegates all long-term storage to `MemoryStore`.
- **Resurrection**: On startup, it loads active conversations from `MemoryStore` into ETS.

### Tracking Levels & Decay
Conversations decay based on silence (minutes since `last_activity`):
- **`:interactive`**: High-priority tool chains or direct user engagement (30m silence -> `:active`).
- **`:active`**: Regular participant (120m silence -> `:watched`).
- **`:watched`**: Observer mode (360m silence -> `:untracked`).
- **`:untracked`**: Minimal metadata tracking (1440m silence -> `:closed` or `:idle`).

### Public Functions
- `lookup/2`: Fast ETS read for `ConversationContext`.
- `touch/3`: Updates activity, increments turns, and recalculates context.
- `get_or_create/3`: Retrieves from ETS, resurrects from `MemoryStore`, or initializes a new conversation.
- `set_tracking_level/4`: Explicitly promotes/demotes tracking.
- `archive_closed/2`: Triggers the compaction and archival pipeline.
- `record_agent_action/3`: Tracks agent participation and promotes tracking level to `:active` if necessary.
- `request_tag/3`: Creates a new tool chain/research thread with `bypass_router: true`.

---

## 4. Compaction

Compaction determines how a conversation is remembered before being purged from the hot path.

### Tier Scoring
The `Myelin.Conversation.Compaction` module calculates a score (0.0 - 1.0) based on:
- **Salience**: Cumulative and peak interest.
- **Tracking Level**: How much attention the agent paid.
- **Participants**: Presence of Tier 1 entities or agent participation.
- **Type**: Technical/Research sessions score higher; shitposting scores lower.
- **Depth**: Turn count as a signal of value.
- **Novelty**: New information relative to existing Engram memories.

### Tiers
1. **`:drop`**: Not worth remembering (Score < 0.2).
2. **`:oneliner`**: Simple metadata store (Score < 0.4).
3. **`:summary`**: Paragraph-length summary (Score < 0.7).
4. **`:deep`**: Detailed briefing + summary (Score >= 0.7).

### Archival Pipeline
When a conversation is archived:
1. Tier and novelty are calculated.
2. If the tier is `:summary` or `:deep` and no summary exists, a **Deferred Summary** is triggered via the Processor.
3. Once the summary is ready, the conversation state is set to `:archived`.
4. The record is persisted to `MemoryStore` (optionally creating an Engram or Briefing).
5. The entry is removed from ETS.

---

## 5. Processor

The `Myelin.Processor` handles asynchronous, model-heavy operations using a local Llama 4B model (default endpoint: `http://localhost:8080`).

### GenServer Design
- **FIFO Queue**: Manages incoming requests with two priorities: `:real` and `:speculative`.
- **Preemption**: If a `:real` task arrives while a `:speculative` task is running, the speculative task is killed (`:brutal_kill`) to ensure low latency for real-time requests.
- **Speculative Scheduler**: When idle, the Processor picks tasks from `SpecScheduler` (e.g., pre-summarizing threads or grooming entities) based on historical hit rate.

### Results
Results are returned via message passing to the `reply_to` pid:
`{:processor_result, operation, request_id, result}`.

---

## 6. Recipes

Recipes are composable, multi-step operations defined in `Myelin.Processor.Recipes`.

| Recipe | Input | Model | Output | Side Effects |
|---|---|---|---|---|
| `search_and_summarize` | `%{query}` | Llama 4B | Summary | Reads from `MemoryStore` |
| `correlate_events` | `%{event}` | Llama 4B | Correlation Brief | Reads related briefings |
| `draft_response` | `%{event}` | Llama 4B | Response Draft | Reads memories for context |
| `deep_entity_profile` | `%{handle}` | Llama 4B | Profile Brief | Reads history from `MemoryStore` |
| `web_fetch_and_store`| `%{url}` | N/A | Fetch Status | Fetches & stores web content |
| `summarize_conversation`| `%{conversation_id}` | Llama 4B | Summary String | Updates `ConversationRegistry` |

---

## 7. TaskScheduler

The `TaskScheduler` manages deferred work that isn't required for immediate response but adds value to long-term memory or future turns.

### Scheduling & Triggers
- **Manual Enqueue**: Other components can call `enqueue/2`.
- **Recurring**: Periodic tasks (e.g., trend detection) scheduled via `schedule_recurring/4`.
- **Cache Window Matching**: The scheduler listens for `cache_signals` (PubSub). When a cache window for a specific topic and tier (Haiku/Sonnet/Opus) becomes available, it attempts to match pending tasks to that window.

### Complexity & Dispatch
- **`:low`**: Dispatched to `Processor.async` with `priority: :speculative`.
- **`:medium` / `:high`**: Dispatched to the `Attentive` state machine for more intensive processing.

---

## 8. Research Threads

Research threads (or tool chains) are special conversations created via `ConversationRegistry.request_tag/3`.

- **`request_tag/3`**: Generates a ULID and initializes a conversation with `:active` tracking.
- **`bypass_router: true`**: This policy override is critical. It signals to the `InferenceRouter` that events associated with this thread should skip the standard two-pass routing logic (vibe check -> action) and be handled directly by the requesting tool or tool chain.

---

## 9. Dependencies

- **ConversationRegistry**: Depends on `MemoryStore` (persistence), `Conversation.Compaction` (archival logic), and `Processor` (async summary).
- **Processor**: Depends on `LlamaClient` (HTTP interface to 4B model), `Operations` (prompt building), and `Recipes` (chaining).
- **TaskScheduler**: Depends on `Processor`, `Budget` (cost gating), `Cache.Manager` (window availability), and `Attentive` (complex task execution).
- **External Services**: Local Llama Endpoint (`:8080`), Engram (`toybox:9005/mcp`).

---

## 10. ETS Tables

- **`:"#{name}_conversations"`**: Owned by `ConversationRegistry`. Stores full `%Conversation{}` structs and `%ConversationContext{}` (as `{:context, thread_id}`).
- **Note**: In the default configuration, `name` is `Myelin.ConversationRegistry`.

---

## 11. Critical Question: Persistence Relationship

**What is the relationship between ConversationRegistry persistence and MemoryStore?**

`ConversationRegistry` **never** writes to SQLite directly. It treats `MemoryStore` as its persistence layer. 
1. When a conversation is "touched" or created, it is marked as **dirty** in the GenServer state.
2. A periodic `persist_dirty` timer (default 30s) flushes all dirty conversations to `MemoryStore.upsert_conversation/2`.
3. On archival or closure, it explicitly calls `MemoryStore.close_conversation/3`.
4. `MemoryStore` is the sole owner of the SQLite connection and schema.

---

## 12. Critical Question: Processor Crash Handling

**What happens to in-flight Processor work if the Processor crashes?**

1. **State Loss**: The `queue` and `current_task` metadata (request_id, reply_to, priority) are stored in the GenServer's memory. If the Processor crashes and restarts, this state is **lost**.
2. **Orphaned Tasks**: The actual work is performed by Elixir `Task` processes spawned via `Task.Supervisor.async_nolink`. If the GenServer crashes, these tasks continue to run to completion, but their results have no listener to receive them (the previous GenServer pid is dead).
3. **Task Failures**: If the *Task* crashes but the *Processor* survives, the Processor catches the `:DOWN` message, notifies the `reply_to` pid with `{:error, reason}`, and proceeds to the next item in the queue.
4. **Preemption**: When a speculative task is preempted, it is terminated via `Task.shutdown(task, :brutal_kill)`. No result is sent.
