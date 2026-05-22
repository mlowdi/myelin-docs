# Design: Pluggable Memory Architecture

*Draft 2026-03-29. Connects to: agent-metabolism-memory-grooming.md, SPEC-interfaces.md*

## Motivation

Myelin's MemoryStore is currently a monolithic GenServer facade over SQLite + ETS + Engram. It conflates three fundamentally different concerns:

1. Sub-millisecond operational lookups that the enrichment pipeline depends on
2. The agent's evolving understanding of its world (personal + world knowledge)
3. Retrieval from external knowledge sources

These have different latency profiles, different lifecycles, different reasons to exist. Separating them makes Myelin a *memory orchestrator* — the agent's core identity isn't tied to any particular storage backend. A deployment can be "the mind of a Kubernetes cluster" where ELK is literally part of the agent's memory, not just an API it can call.

## Three-layer model

The layers are distinguished by **agency** — who owns the data and who manages its lifecycle:

| Layer | Automatic mgmt | Agent mgmt | External mgmt |
|-------|---------------|------------|---------------|
| **Operational state** | Always | Sometimes | Never |
| **Agent memory** | Sometimes (grooming, summaries) | Always | Never |
| **Knowledge providers** | Never | Sometimes | Sometimes (or static) |

Operational state is the autonomic nervous system — it runs without the agent thinking about it, though the agent *can* intervene (ignore an entity, register a hot topic). Agent memory is the agent's mind — it always owns what it remembers, even if automated processes help maintain it. Knowledge providers may be curated by the agent, managed externally (an ops team maintaining a wiki), or entirely static (a reference corpus). The agent queries them but doesn't necessarily control their contents.

### Layer 1: Operational State

**What**: Entity cache, conversation tracking, session state, hot topics, failure log, active rules.

**Character**: Fast, structural, internal. The enrichment hot path reads this on every event. Latency budget: sub-millisecond for reads.

**Agency**: Always automatically managed (ETS hydration, cache invalidation, timer-based decay). The agent may *also* manage it explicitly (ignore an entity, register a hot topic, insert a rule) but doesn't have to — the system runs autonomously.

**Implementation**: ETS for reads, SQLite for durability. Hardwired — not pluggable. This is Myelin's autonomic nervous system; it doesn't change based on deployment.

**Stays in**: `Myelin.MemoryStore` (or a renamed `Myelin.OpState` / `Myelin.Registry`).

**API** (unchanged from today):
- `lookup_entity/2`, `upsert_entity/2`, `touch_entity/2`, `ignore_entity/3`
- `upsert_conversation/2`, `lookup_conversation/2`, `list_active_conversations/1`
- `upsert_session/2`, `lookup_session/2`, `close_session/4`
- `register_hot_topic/3`, `check_hot_topics/2`
- `match_keywords/2`, `get_active_rules/1`, `insert_rule/2`
- `record_failure/2`, `failure_summary/2`

### Layer 2: Agent Memory

**What**: The agent's personal experience and world knowledge. "Things I've learned" and "things I know about the world." Fed by inference — when the agent processes events, routes messages, generates responses, it may produce memories.

**Character**: Dynamic, semantic, groomable. Needs consistent interface regardless of backing store. One active backend per deployment (but swappable).

**Agency**: Always agent-managed — the agent decides what to remember, what to forget, what to search for. Automatic processes (grooming, storing conversation summaries, inference-triggered memory creation) *assist* but the agent is the owner. No external system writes to agent memory.

**Behaviour**: `Myelin.Memory`

```elixir
defmodule Myelin.Memory do
  @doc "Store a memory unit with metadata (source, timestamp, kind hint)."
  @callback store(content :: String.t(), metadata :: map()) ::
    {:ok, id :: String.t()} | {:error, term()}

  @doc "Semantic search over stored memories."
  @callback search(query :: String.t(), opts :: keyword()) ::
    {:ok, [memory()]} | {:error, term()}

  @doc "Serendipity function — surface a random/surprising memory.
  The point is re-encounter, not retrieval. Let the agent stumble
  on something it wasn't looking for."
  @callback random_recall(opts :: keyword()) ::
    {:ok, [memory()]} | {:error, term()}

  @doc "Mark a memory for grooming evaluation. The grooming system
  will batch-evaluate marked memories using a cheap model."
  @callback mark_for_evaluation(id :: String.t(), reason :: atom()) ::
    :ok | {:error, term()}

  @doc "Retrieve memories currently marked for evaluation."
  @callback pending_evaluation(opts :: keyword()) ::
    {:ok, [memory()]} | {:error, term()}

  @doc "Execute a grooming action (merge, delete, update)."
  @callback groom(action :: grooming_action()) ::
    :ok | {:error, term()}

  @type memory :: %{
    id: String.t(),
    content: String.t(),
    metadata: map(),
    score: float() | nil,
    stored_at: DateTime.t()
  }

  @type grooming_action ::
    {:delete, id :: String.t()}
    | {:merge, source_ids :: [String.t()], merged_content :: String.t(), metadata :: map()}
    | {:update, id :: String.t(), content :: String.t(), metadata :: map()}
end
```

**Key design choices**:
- No pre-categorization at write time. Personal vs world knowledge is a grooming-time distinction, not a storage-time one.
- `random_recall` is a first-class operation, not an afterthought. Serendipitous re-encounter is how the agent develops intuition.
- `mark_for_evaluation` is the bridge to the metabolism system. Threshold-triggered detection (query returned too many results, registry too large) marks candidates; the grooming evaluator (Haiku) processes them in batches.
- Metadata is intentionally unstructured (`map()`). Different backends will want different metadata. The behaviour doesn't prescribe schema.

**Implementations**:
- `Myelin.Memory.Engram` — current default. Semantic vector search via Engram FastMCP server.
- `Myelin.Memory.SQLite` — fallback for deployments without a vector DB. LIKE-based search, random recall via `ORDER BY RANDOM()`.
- Future: Letta (git-backed), pgvector, Qdrant, etc.

**Configuration**: Single backend selected in application config or `config/memory.toml`:
```toml
[memory]
backend = "engram"

[memory.engram]
url = "http://localhost:3001"
collections = ["personal", "world"]
```

### Layer 3: Knowledge Providers

**What**: External knowledge sources the agent can query. Research corpora, monitoring systems, document stores, APIs, filesystems. These aren't "the agent's memories" in the experiential sense — they're reference material. But they're accessed *as memory*, not as tools.

**Character**: Slow (10ms–10s), async-tolerant, read-heavy, externally populated. Zero, one, or many providers active simultaneously. Each provider may have its own auth, rate limits, connection lifecycle.

**Agency**: Maybe agent-managed (the agent curates a research corpus), maybe externally managed (an ops team maintains a runbook wiki, a CI pipeline updates a docs index), maybe static (a reference dataset that never changes). The agent queries but doesn't necessarily own or control the contents. This is the key distinction from Layer 2 — agent memory is *always* the agent's; knowledge may belong to someone else entirely.

**The insight**: The difference between "I can search the logs" (tool) and "the logs are part of my memory" (knowledge provider) is *how the agent relates to the results*. A tool returns data. A memory provider returns something the agent *already knew but needed to recall*. This changes how the agent reasons about the results — they carry epistemic weight.

**Behaviour**: `Myelin.Knowledge`

```elixir
defmodule Myelin.Knowledge do
  @doc "Query this knowledge source. Returns ranked results."
  @callback query(query :: String.t(), opts :: keyword()) ::
    {:ok, [result()]} | {:error, term()}

  @doc "Describe what this provider knows about. Used by the router
  to decide which providers are relevant to a query."
  @callback scope() :: String.t()

  @doc "Health/availability check."
  @callback available?() :: boolean()

  @type result :: %{
    content: String.t(),
    source: String.t(),
    relevance: float() | nil,
    metadata: map()
  }
end
```

**Key design choices**:
- `scope/0` is how the system knows *when* to query a provider. The router doesn't fan out to every provider on every query — it matches query intent against provider scope descriptions. This is itself an LLM-evaluable decision.
- Knowledge providers are stateless from Myelin's perspective. They don't need grooming. They don't accumulate. They're windows into external state.
- Async by nature — queries may return via callback or be fire-and-forget with results arriving later. The behaviour shows the sync interface; an async wrapper is a separate concern.

**Implementations** (examples):
- `Myelin.Knowledge.Filesystem` — `find`/`grep` over a directory tree. Scope: "local files and documents."
- `Myelin.Knowledge.ELK` — Elasticsearch/OpenSearch queries. Scope: "application logs and metrics." The k8s agent's memory of what happened in the cluster.
- `Myelin.Knowledge.ClickHouse` — analytics queries. Scope: "historical metrics and events."
- `Myelin.Knowledge.RAG` — generic HTTP RAG API adapter. Scope: configured per instance.
- `Myelin.Knowledge.Web` — web search integration. Scope: "public internet."

**Configuration**: Multiple providers via `config/knowledge/*.toml` (same pattern as interfaces):
```toml
# config/knowledge/elk.toml
[provider]
type = "elk"
scope = "Kubernetes cluster logs, pod events, and deployment history"

[connection]
url = "https://elk.internal:9200"
index_pattern = "k8s-*"

[secrets]
api_key = "ELK_API_KEY"
```

**Loaded by**: `Myelin.Knowledge.Loader` — same `DynamicSupervisor` pattern as `Interface.Loader`. Hot-loadable.

## How the layers interact

```
Event arrives
    |
    v
[Layer 1: OpState] -- sub-ms entity lookup, keyword match, hot topics
    |
    v
EventPipeline scores, routes
    |
    v
Agent processes event (inference)
    |
    +---> [Layer 2: Agent Memory] -- "what do I remember about this?"
    |         search(), random_recall()
    |
    +---> [Layer 3: Knowledge Providers] -- "what do I know about this domain?"
    |         query() on relevant providers (selected by scope matching)
    |
    v
Agent generates response
    |
    +---> [Layer 2: Agent Memory] -- store() new memories from this interaction
    |
    +---> [Layer 1: OpState] -- upsert_entity, upsert_conversation, etc.
    |
    v
Response delivered
```

The grooming system operates on Layer 2 independently:
```
Threshold detector (mechanical)
    |
    v
mark_for_evaluation() on candidates
    |
    v
Grooming evaluator (Haiku, batched, cached context)
    |
    v
groom({:delete, id}) | groom({:merge, ids, content}) | defer + flag
```

## Design decisions (2026-03-29)

- **Dual-write dies.** The current pattern of writing briefings to both SQLite and Engram is a holdover. Memory backends own their own durability. No mirroring.
- **Briefings are NOT agent memory.** They're system-generated research artifacts (even when agent-requested). Ontologically closer to operational state than to the agent's mind. They stay in Layer 1 / MemoryStore, not the `Myelin.Memory` behaviour. The functions that move to Layer 2 are: `store_engram`, `search_engrams`, `random_recall` — the actual semantic memory operations.
- **Engram is a placeholder, not the architecture.** It was built as a personal assistant's semantic memory MCP server — "the cheapest thing lying around." The `Memory` behaviour is what matters. Engram implements it today. An Elixir-native semantic store replaces it in the final cohesive rewrite. Don't build around Engram's quirks.
- **Move fast, fix with workers.** No facade delegation phase. Cut the module, migrate callers, dispatch Gemini workers on anything that breaks. Code is cheap.

## Migration path

Direct cut, not incremental facade. Workers handle breakage in parallel.

### Wave 1: Behaviours + extraction — DONE (2026-03-29, PR #145)
- `Myelin.Memory` behaviour with store/search/random_recall/mark_for_evaluation/groom callbacks
- `Myelin.Knowledge` behaviour with query/scope/available? callbacks
- `Myelin.Memory.Engram` implementation wrapping EngramClient
- Dispatch module in `Myelin.Memory` delegating to configured backend

### Wave 2: Caller migration — DONE (2026-03-29, PR #146 + direct commit)
- Migrated `Processor.Recipes`, `ConversationRegistry`, `ResearchThread.Tools` to `Myelin.Memory`
- Removed `store_engram/3`, `search_engrams/4`, `random_recall/3` from MemoryStore
- Removed `engram_disabled` flag, dual-write from `insert_briefing`, Engram fallback from `search_briefings`
- MemoryStore lost 96 lines and all Engram coupling

### Wave 3: Knowledge providers — DONE (2026-03-29, PRs #148, #149, #150)
- `Myelin.Knowledge.Supervisor` (DynamicSupervisor + Registry)
- `Myelin.Knowledge.Loader` — TOML-driven, same pattern as Interface.Loader
- `Myelin.Knowledge.Local` wrapping existing KnowledgeStore FTS5/Vec0
- `knowledge_query` tool wired into research thread
- `query_all/2` fan-out dispatch to all registered providers

### Wave 4: Grooming — DONE (PRs #156, #157)
Prerequisites now met:
- `Myelin.Memory` behaviour is live with `mark_for_evaluation`, `pending_evaluation`, `groom` callbacks
- Inference tier system is live (PRs #151-153) — grooming evaluator requests `min_tier: :capable` instead of hard-coding "use Haiku"
- `random_recall` exists in the behaviour but has zero callers

Steps:
12. **Threshold detector** — mechanical module that checks counts/sizes and calls `Myelin.Memory.mark_for_evaluation`:
    - Memory search returns > N results → flag for merge evaluation
    - Entity registry > threshold → flag entity candidates
    - Rules registry > 20 active → flag for evaluation
13. **Grooming evaluator** — batched evaluation using `min_tier: :capable`:
    - Pulls `pending_evaluation` batch
    - Constructs evaluation prompt: "are any of these mergeable / should any be removed?"
    - Executes grooming actions via `Myelin.Memory.groom`
    - If "nah, all different" → set cooldown flag, recheck after N hours
14. **Wire `random_recall`** into agent context assembly (serendipity function):
    - Add to context builder as an optional component
    - Trigger probabilistically or on low-activity periods
    - Surface as "something you remembered" in the agent's context

## What this enables

- **"Mind of a K8s cluster"**: ELK + Prometheus as knowledge providers. The agent doesn't "query logs" — it *remembers* what happened in the cluster. The epistemic framing changes how it reasons.
- **Memory backend flexibility**: Swap Engram for pgvector, Qdrant, or a native Elixir implementation without touching any caller code. Local-only deployments use SQLite fallback.
- **Composable agent personalities**: Same core, different memory configurations. A security agent with Splunk as memory. A support agent with Zendesk ticket history as memory. A developer agent with git log + filesystem as memory.
- **Grooming that scales**: The metabolism system operates on the Memory behaviour, not on SQLite directly. Whatever backend stores the memories, the same threshold-triggered evaluation handles grooming. Evaluator requests `min_tier: :capable` via the inference tier system (DESIGN-inference-tiers.md) — no hard-coded model names.
- **Briefings as first-class operational data**: No longer conflated with agent memory. System-generated research artifacts have their own lifecycle, queryable but not groomable.

## Open questions

- Should knowledge providers support write operations? (e.g., the agent writing back to a wiki, updating a runbook). Current design is read-only — writes go through the tool system or output delivery.
- How does the router decide which knowledge providers are relevant? Embedding-based scope matching vs. keyword heuristic vs. LLM judgment call. Probably starts as LLM judgment, optimizes later.
- Should Layer 2 support multiple concurrent backends? (e.g., semantic + structured). Current design says one — but the dispatch layer could compose if needed.
- When we rewrite Engram in Elixir, what's the minimal viable semantic search? SQLite FTS5 + simple embedding similarity, or do we need a proper vector index?
