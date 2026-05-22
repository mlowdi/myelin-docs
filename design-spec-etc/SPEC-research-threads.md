# Research Threads — Attention-Driven Async Research

*With cached identity, Haiku calls cost ~$0.0008. That's cheap enough to let Haiku follow leads, run searches, synthesize briefings, and store results — all async, all attention-driven, all budget-capped.*

---

## The New Haiku Role

The original design positioned Haiku as a gate: assemble context, assess confidence, escalate or resolve. One-shot. The layered cache strategy changes the economics enough to expand this:

```
Before (Haiku as gate):
  Event → context assembly → Haiku call → resolve or escalate
  Role: single-event assessment

After (Haiku as research layer):
  Event → context assembly → Haiku call → resolve, escalate, OR research
  Interesting signal → Haiku research thread → tool calls → briefing → store
  Role: assessment + autonomous research with tool access
```

Haiku can now:
- Run web searches (via tool calls)
- Search Bluesky for related discussions
- Query engram for related memories
- Synthesize findings into a structured briefing
- Store the briefing for Sonnet to reference later

All of this happens async, with a token budget, and the results fold into a searchable store.

---

## Research Thread Lifecycle

```
     interesting signal detected
     (Processor trend detection, router "watch" decision,
      Sonnet explicit request, or Haiku's own assessment)
              │
              ▼
     ┌─────────────────┐
     │ RESEARCH THREAD  │
     │ CREATED          │
     │                  │
     │ topic: ...       │
     │ budget: N tokens │
     │ priority: ...    │
     │ source: ...      │
     └────────┬─────────┘
              │
              │  scheduled by TaskScheduler
              │  (rides a cache window, or gets its own cheap Haiku call)
              ▼
     ┌─────────────────┐
     │ RESEARCHING      │ ← Haiku making tool calls
     │                  │   web search, bsky search, engram query
     │ tokens_used: ... │   each call eats from the budget
     │ findings: [...]  │
     └────────┬─────────┘
              │
              │  budget exhausted, or Haiku signals "done"
              ▼
     ┌─────────────────┐
     │ SYNTHESIZING     │ ← Haiku produces a structured briefing
     │                  │   from accumulated findings
     └────────┬─────────┘
              │
              ▼
     ┌─────────────────┐
     │ STORED           │ ← briefing saved to engram collection
     │                  │   searchable by topic/entity/keyword
     │ briefing_id: ... │   available for Sonnet's frozen/hot zone
     └─────────────────┘
```

---

## Research Thread Schema

```elixir
defmodule AgentRuntime.ResearchThread do
  @type t :: %__MODULE__{
    id: String.t(),

    # what and why
    topic: String.t(),                  # what we're researching
    topic_embedding: [float()],         # for semantic matching
    trigger: atom(),                    # :trend | :watch | :sonnet_request | :haiku_curiosity
    trigger_ref: String.t() | nil,      # event/session/rule that spawned this

    # budget
    token_budget: non_neg_integer(),    # max tokens to spend (input + output)
    tokens_used: non_neg_integer(),     # running total
    max_tool_calls: non_neg_integer(),  # cap on tool invocations (default: 10)
    tool_calls_made: non_neg_integer(),

    # state
    state: :pending | :researching | :synthesizing | :stored | :abandoned,
    priority: float(),                  # for TaskScheduler ordering

    # findings (accumulated during research)
    findings: [%{
      source: atom(),                   # :web_search | :bsky_search | :engram | :rss
      query: String.t(),
      result_summary: String.t(),       # Haiku-compressed result
      relevance: float(),               # Haiku's assessment
      timestamp: DateTime.t()
    }],

    # output
    briefing: String.t() | nil,         # final synthesized briefing
    briefing_tokens: non_neg_integer(), # how big is the briefing
    engram_collection: String.t(),      # which engram collection to store in
    engram_ids: [String.t()] | nil,     # memory IDs created from briefing

    # lifecycle
    created_at: DateTime.t(),
    ttl: non_neg_integer(),             # seconds before abandonment
    completed_at: DateTime.t() | nil
  }
end
```

---

## Triggers — What Starts a Research Thread

### From Processor (speculative)
Processor's `:trend_detection` notices: "umbra has posted 4 times about formation-honesty in 6 hours." Creates a research thread: "What is formation-honesty? Search web, search Bluesky, check engram for related discussions." Priority: low. Budget: 5,000 tokens. When Haiku has a warm cache and idle capacity, it picks this up.

### From Router ("watch" decision)
Router's Pass 2 returns `:watch` for an event. If the topic is novel (no engram hits, no hot topic match), optionally spawn a low-priority research thread: "This topic came up and we know nothing about it. Background research." Budget: 3,000 tokens.

### From Sonnet (explicit request)
Sonnet, during an Engaged session, can request research via tool call: `request_research(topic: "credential stuffing techniques 2026", budget: 10000, priority: :high)`. Gets picked up next time Haiku has capacity. Results available for Sonnet's next Engaged session.

### From Haiku (self-directed)
During an Attentive assessment, Haiku finds something interesting but doesn't need to escalate to Sonnet. Instead of just resolving, it spawns a research thread for itself: "I resolved this event but the topic seems worth tracking. Research when free." Self-sustaining curiosity.

---

## Tool Access

Research threads give Haiku access to a constrained set of tools:

```elixir
@research_tools [
  :web_search,           # search the web via API
  :bsky_search,          # search Bluesky posts/threads
  :engram_query,         # semantic search against memory
  :rss_digest,           # pull pre-digested RSS briefs (from Processor speculative work)
  :fetch_url             # fetch and digest a specific URL
]

# NOT available to research threads:
# :post_to_bsky        — research is read-only
# :write_rule          — only Sonnet can reprogram routing
# :modify_entity       — only Sonnet can change entity tiers
```

Research threads are **read-only**. They gather and synthesize. They never act. Action is Sonnet's domain.

---

## Budget Controls

```elixir
defmodule AgentRuntime.ResearchThread.Limits do
  @max_concurrent_threads 5            # don't hog Haiku capacity
  @max_daily_research_budget 50_000    # tokens/day across all threads
  @max_per_thread_budget 15_000        # no single thread runs away
  @max_tool_calls_per_thread 10        # cap on API tool invocations
  @default_thread_ttl 6 * 3600         # 6 hours before abandonment
  @default_priority 0.3                # low by default, Sonnet requests get 0.8+
end
```

Research budget is tracked separately from the main API budget. It's carved out of the daily budget but doesn't compete with real-time escalation — if the main budget is tight, research threads pause.

---

## Storage — "Stuff I Read Online"

Research briefings go into an engram collection (or collections). Simple semantic memory:

- **Collection per topic domain** (optional): `research:security`, `research:identity`, `research:elixir`, or just a flat `research` collection
- **Each briefing** is a memory with: topic, source summary, key claims, entities mentioned, relevance assessment, timestamp
- **Searchable** by semantic similarity — when Sonnet enters Engaged, the context assembler can pull relevant research briefings into the hot zone

This is literally just engram with a namespace. The engram MCP server already supports collections. No new infrastructure needed — just a convention for how research threads store their output.

### Research → Sonnet pipeline

```
Haiku research thread stores briefing in engram
                    │
                    ▼
        engram collection: "research"
                    │
    ┌───────────────┴───────────────┐
    │                               │
    ▼                               ▼
Sonnet Engaged session          Haiku Attentive session
(context assembler pulls        (can reference for richer
 relevant research into          context assembly)
 hot zone or ephemeral)
```

When Sonnet enters Engaged about a topic, the context assembler does an engram search. If there's a research briefing about that topic, it goes into the hot zone: "Background research available: [briefing]." Sonnet arrives pre-briefed.

---

## Integration with Existing Architecture

### TaskScheduler
Research threads are a new deferred task type. They ride cache windows like any other deferred work. A warm Haiku cache with remaining TTL is a perfect opportunity to run a pending research thread — the identity layer is already paid for, just add the research context in the hot zone.

### Speculative Computation
Research threads are speculative by nature. They fit naturally into the Processor's quiet-hours saturation strategy — but they run on Haiku (API) instead of Processor (local 4B). The scheduling logic is the same: pick highest priority pending thread, check budget, dispatch.

### Sessions
Research threads are NOT sessions. They don't escalate, they don't trigger state transitions, they don't write rules. They're background processes that produce briefings. A research thread might be *spawned by* a session (Sonnet request) but it runs independently.

### Cache Layers
Research threads benefit maximally from the layered cache. Layer 0+1 are already warm → research thread pays only for Layer 2 (if needed) + hot zone (research context + tool results). Marginal cost per research step: ~$0.001-0.003.

---

## Design Properties

- **Read-only.** Research threads gather information. They never act, post, or modify system state (beyond storing their output in engram).
- **Budget-capped.** Every thread has a token budget. When it's spent, synthesize and store whatever you have. No runaway costs.
- **Attention-driven.** Threads are spawned because the system noticed something interesting. Not scheduled — triggered by salience signals.
- **Self-sustaining.** Haiku can spawn its own research threads. The system can be curious without Sonnet's involvement. Most research is low-priority background work that runs whenever there's spare capacity.
- **Results are memories.** Research output goes into engram, making it available to all future inference calls. The system literally gets smarter over time — not through training, but through accumulated research.
