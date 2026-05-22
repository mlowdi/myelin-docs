

# Cache Strategy --- Context Zones and Opportunistic Filling

*Prompt caching isn't "save money on repeated prompts." It's "decide
which parts of your mind are permanent and which are ephemeral, and only
pay for the permanent parts once."*

------------------------------------------------------------------------

## How Anthropic Prompt Caching Works

-   Place `cache_control: {"type": "ephemeral"}` breakpoints in messages
-   Everything before the breakpoint is cached for 5 minutes
    (auto-extended on hit)
-   Cache write: 1.25x base input price (small premium, paid once)
-   Cache read: 0.1x base input price (90% discount on subsequent calls)
-   Up to 4 breakpoints available

The game: put the stable stuff before the breakpoint, the changing stuff
after. The more you freeze, the less you pay per call.

------------------------------------------------------------------------

## Context Zone Architecture

    ┌─────────────────────────────────────────────────────┐
    │ FROZEN ZONE                                         │
    │                                                     │
    │ ┌─────────────────────────────────────────────────┐ │
    │ │ System Identity                                 │ │
    │ │ - personality spec (core tier, ~2200 tokens)    │ │
    │ │ - relationship context (compressed, ~500 tokens)│ │
    │ │ - voice/tone rules (~300 tokens)                │ │
    │ │ - hard behavioral constraints (~200 tokens)     │ │
    │ └─────────────────────────────────────────────────┘ │
    │                                                     │
    │ ┌─────────────────────────────────────────────────┐ │
    │ │ Active Rules Snapshot                           │ │
    │ │ - current state machine state + policy          │ │
    │ │ - active rules table (serialized, ~200-500 tok) │ │
    │ │ - entity registry top-tier entries (~300 tokens) │ │
    │ └─────────────────────────────────────────────────┘ │
    │                                                     │
    │ ┌─────────────────────────────────────────────────┐ │
    │ │ Session Memory                                  │ │
    │ │ - compressed conversation history               │ │
    │ │ - hot topics register snapshot                  │ │
    │ │ - recent Processor outputs (key summaries only) │ │
    │ └─────────────────────────────────────────────────┘ │
    │                                                     │
    │              ═══ CACHE BREAKPOINT ═══               │
    ├─────────────────────────────────────────────────────┤
    │ HOT ZONE (register — rewritten each call)           │
    │                                                     │
    │ - current event being processed                     │
    │ - Processor results for THIS event                  │
    │ - thread context (compressed by Processor)          │
    │ - relevant engram hits (summarized by Processor)    │
    │ - specific instructions for this inference          │
    │                                                     │
    ├─────────────────────────────────────────────────────┤
    │ EPHEMERAL (use and discard)                         │
    │                                                     │
    │ - raw tool results                                  │
    │ - intermediate reasoning                            │
    │ - draft outputs being evaluated                     │
    │ - full payload content (if fetched)                 │
    │                                                     │
    └─────────────────────────────────────────────────────┘

------------------------------------------------------------------------

## Token Budgets

Hard limits enforced by pre-flight token counting before every API call.
**The frozen zone expands on escalation** --- each tier gets a richer
frozen context because higher-tier models need more context to do their
job well.

``` {.sourceCode .elixir}
defmodule AgentRuntime.Cache.Budget do
  # Frozen zone scales with tier — more expensive models get richer identity
  @frozen_budget %{
    attentive: %{
      identity: 2200,        # core personality tier only
      relationship: 500,     # compressed
      voice_constraints: 500,
      rules_snapshot: 800,
      session_memory: 1000,  # aggressive compression
      total: 5000
    },
    engaged: %{
      identity: 4400,        # core + rich personality tiers
      relationship: 800,     # full, less compressed
      voice_constraints: 500,
      rules_snapshot: 800,
      entity_registry: 500,  # top-20 entities with context
      session_memory: 2000,  # richer history
      hot_topics: 500,       # detailed topic context, not just scores
      total: 9500
    },
    interactive: %{
      identity: 4400,        # core + rich
      relationship: 800,
      voice_constraints: 500,
      rules_snapshot: 800,
      entity_registry: 500,
      session_memory: 2000,
      hot_topics: 500,
      human_preferences: 500,   # Martin's current state, recent feedback
      project_context: 500,     # what we're working on
      total: 10500
    }
  }

  @hot_budget %{
    attentive: 2000,       # Haiku gets modest hot zone
    engaged: 4000,         # Sonnet gets richer hot zone
    interactive: 6000      # human conversation gets the most room
  }

  @ephemeral_budget %{
    attentive: 1000,       # Haiku: tight ephemeral budget
    engaged: 4000,         # Sonnet: can work with more raw material
    interactive: 8000      # interactive: room for tool results, drafts
  }

  # Total per-call ceiling:
  # Attentive (Haiku):  ~8k tokens  (5k frozen + 2k hot + 1k ephemeral)
  # Engaged (Sonnet):   ~17.5k tokens (9.5k frozen + 4k hot + 4k ephemeral)
  # Interactive:         ~24.5k tokens (10.5k frozen + 6k hot + 8k ephemeral)
end
```

### Frozen zone expansion on escalation

The frozen zone grows when the system escalates. Each tier builds a
different frozen context:

``` {.sourceCode .elixir}
defp build_frozen_zone(:attentive) do
  [
    load_personality(:core),           # core tier only (~2200 tokens)
    compress(relationship_context()),  # tight compression
    voice_and_constraints(),
    rules_snapshot(),
    compress(session_memory(), 1000)   # aggressive compression
  ]
end

defp build_frozen_zone(:engaged) do
  [
    load_personality(:full),           # core + rich tiers (~4400 tokens)
    relationship_context(),            # full, less compressed
    voice_and_constraints(),
    rules_snapshot(),
    entity_registry(:top_20),          # more entities visible
    session_memory(),                  # richer history
    hot_topics_detailed()              # topic context, not just scores
  ]
end

defp build_frozen_zone(:interactive) do
  build_frozen_zone(:engaged) ++ [
    human_preferences(),               # Martin's current state, recent feedback
    active_project_context()           # what we're working on
  ]
end
```

The cost of expansion: `(new_frozen_tokens - old_frozen_tokens) * 1.25x`
on the first call to the new tier. Every subsequent call reads the
*entire expanded block* at 0.1x. Cache write pays for itself on the
second call.

De-escalation is free --- just let the cache expire (5-min TTL). Next
escalation rebuilds the frozen zone fresh with current state. Natural
garbage collection.

------------------------------------------------------------------------

## Cache Window Lifecycle

``` {.sourceCode .elixir}
defmodule AgentRuntime.Cache.Manager do
  defmodule Window do
    @type t :: %__MODULE__{
      id: String.t(),
      tier: :attentive | :engaged,
      frozen_hash: String.t(),       # hash of frozen zone — detect staleness
      created_at: DateTime.t(),
      last_hit_at: DateTime.t(),     # resets the 5-min TTL
      expires_at: DateTime.t(),      # 5 min after last hit
      topic_signature: [String.t()], # what this cache window is "about"
      token_usage: %{frozen: integer(), hot: integer(), ephemeral: integer()},
      hit_count: non_neg_integer()
    }
  end
end
```

### Key operations

**`touch_window/1`** --- called after every API inference. Updates
`last_hit_at`, extends expiry by 5 minutes. Keeps the cache alive during
active use.

**`release_window/1`** --- called when an Engaged process goes quiet.
Cache is still warm but nobody's using it. If \>60s remaining, emit
`{:cache_available, ...}` to TaskScheduler for opportunistic use.

**`check_staleness/1`** --- compare frozen zone hash to current state.
If rules or hot topics changed significantly, the frozen zone needs
rewriting (cache write at 1.25x instead of read at 0.1x). Only rewrite
if expected remaining hits justify the cost.

------------------------------------------------------------------------

## TaskScheduler --- Opportunistic Cache Filling

When a cache window has unused TTL remaining, TaskScheduler matches
deferred work by semantic similarity and dispatches it. Zero additional
cache cost for the frozen zone.

### Deferred task types

-   **Memory compaction** --- old session context → engram entries. Can
    ride any cache window
-   **Entity grooming** --- tier review, salience updates. Matches
    social interaction windows
-   **Draft posts** --- blog drafts, queued social posts. Matches
    topical windows
-   **Research queries** --- "next time we're thinking about X, also
    look into Y"
-   **Thread catchup** --- monitored threads not needing real-time
    response. Perfect for leftover topical windows
-   **Rule analysis** --- review rule effectiveness. Matches any window

### Matching algorithm

``` {.sourceCode .elixir}
def handle_info({:cache_available, opts}, state) do
  window = Cache.Manager.get_window(opts.window_id)
  available_tokens = Budget.hot_budget(window.tier) + Budget.ephemeral_budget(window.tier)

  candidates = state.deferred_queue
    |> Enum.filter(fn task ->
      task.min_tier <= window.tier and
      task.token_estimate <= available_tokens and
      not expired?(task)
    end)
    |> Enum.sort_by(fn task ->
      similarity = semantic_similarity(task.topic_signature, window.topic_signature)
      similarity * task.priority - task.token_estimate * 0.001
    end, :desc)

  case candidates do
    [best | _] -> dispatch_to_inference(best, window)
    [] -> :noop  # nothing matches, cache expires unused. that's fine.
  end
end
```

### Economics

Example: Sonnet finishes an Engaged session about a Bluesky thread on
discontinuous identity. Cache has 3 minutes left. TaskScheduler finds
deferred task: "compact today's conversation about identity into engram
memories." Topic match is high. Dispatch it. Sonnet does the compaction
using the already-paid-for frozen zone. Marginal cost: just the
hot+ephemeral tokens.

Over a day, 5-10 opportunistic fills save 30k-60k tokens of frozen-zone
costs. At Sonnet pricing (\$3/M input), that's \$0.09-0.18/day ---
potentially 30-50% of the daily budget.

------------------------------------------------------------------------

## Pre-Flight Token Management

Every API call goes through pre-flight: count tokens (free API), enforce
budgets, compress if needed.

``` {.sourceCode .elixir}
defmodule AgentRuntime.Cache.PreFlight do
  def prepare_context(frozen_zone, hot_zone, ephemeral, tier) do
    budget = %{
      frozen: Budget.frozen_total(),
      hot: Budget.hot_budget(tier),
      ephemeral: Budget.ephemeral_budget(tier)
    }

    counts = %{
      frozen: count_tokens(frozen_zone),
      hot: count_tokens(hot_zone),
      ephemeral: count_tokens(ephemeral)
    }

    # compress if over budget, starting with least valuable zone
    cond do
      counts.ephemeral > budget.ephemeral ->
        # truncate ephemeral (disposable by definition)
        truncate_ephemeral(frozen_zone, hot_zone, ephemeral, budget)

      counts.hot > budget.hot ->
        # ask Processor to rewrite hot zone more tightly
        compress_hot(frozen_zone, hot_zone, ephemeral, budget)

      counts.frozen > budget.frozen ->
        # trim session_memory first (least stable part of frozen zone)
        trim_frozen(frozen_zone, hot_zone, ephemeral, budget)

      true ->
        assemble_prompt(frozen_zone, hot_zone, ephemeral)
    end
  end
end
```

### Compression priority

When over budget, sacrifice in this order: 1. **Ephemeral** --- truncate
freely, it's disposable 2. **Hot zone** --- ask Processor to rewrite
more tightly (async, but worth it) 3. **Frozen session_memory** --- trim
oldest entries (identity and rules_snapshot are never trimmed)

Identity never gets compressed. Rules snapshot rarely changes. Session
memory is the pressure valve.

------------------------------------------------------------------------

## Cache Tiers and State Machine Integration

  --------------------------------------------------------------------------------
  State         Cache Tier       Write Cost        Read Cost        When
  ------------- ---------------- ----------------- ---------------- --------------
  Dormant       None             ---               ---              No API calls,
                                                                    no cache

  Monitoring    None             ---               ---              Local models
                                                                    only, no API
                                                                    calls

  Attentive     Short (5-min)    1.25x on first    0.1x on          Natural social
                                 Haiku call        subsequent       media thread
                                                                    rhythm

  Engaged       Long (1-hour\*)  1.25x on first    0.1x on          Sustained
                                 Sonnet call       subsequent       engagement,
                                                                    multiple calls

  Interactive   Long (1-hour\*)  1.25x on session  0.1x for         Human
                                 start             duration         conversation
  --------------------------------------------------------------------------------

\*"1-hour" = maintained by repeated touches extending the 5-min TTL. As
long as we hit the cache at least once every 5 minutes, it stays alive.

The 5-minute auto-extend means Attentive mode's "short cache" is
actually just "we don't try to keep it alive" --- if Haiku resolves in
one call, the cache expires. If it takes three calls over 10 minutes,
the cache serves all three.

Engaged mode's "long cache" is actively maintained --- TaskScheduler may
dispatch opportunistic work specifically to keep a valuable cache window
alive if deferred tasks match its topic.
