# Cache Strategy v2 — Layered Cache, Decoupled from Sessions

*The frozen zone isn't a session artifact. It's a shared cognitive substrate that multiple sessions ride on. With 4 cache breakpoints and mixed TTLs, we can layer it by stability and amortize the identity cost across an entire day.*

**Supersedes:** SPEC-cache-strategy.md (original single-breakpoint design)

---

## The Key Insight

The Anthropic API input is one string. Anything before a cache breakpoint is frozen. We get 4 breakpoints with mixed TTLs (1-hour at 2x write, 5-min at 1.25x write, both read at 0.1x). The 1-hour cache auto-refreshes on every hit.

This means: if any part of the system makes at least one API call per hour, the 1-hour cached content stays alive **all day**. We write it once (2x cost), then read it potentially dozens of times (0.1x each). The identity layer — personality, voice, constraints — is ~4400 tokens that literally never changes within a day. That's the perfect candidate for a 1-hour cache that lives forever through auto-refresh.

The old design treated the frozen zone as one block owned by the session. The new design splits it into **layers of decreasing stability**, each with its own cache lifecycle, independently reusable by any session.

---

## Anthropic API Cache Constraints

From the API docs (verified 2026-03-13):

| Constraint | Value |
|---|---|
| Max breakpoints | 4 per request |
| Min cacheable tokens (Haiku 4.5) | 4,096 |
| Min cacheable tokens (Sonnet 4.6) | 2,048 |
| 5-min cache write cost | 1.25x base input |
| 1-hour cache write cost | 2.0x base input |
| Cache read cost (either TTL) | 0.1x base input |
| TTL auto-refresh | On every cache hit, no additional cost |
| TTL ordering rule | 1-hour entries MUST come before 5-min entries |
| Cache isolation | Workspace-level |

**Critical detail:** 1-hour entries must appear before 5-min entries in the prompt. This constrains our layer ordering — the most stable (1-hour) layers go first, the more volatile (5-min) layers follow. Which is exactly what we want anyway.

---

## Layered Frozen Zone Architecture

Instead of one frozen zone with one breakpoint, we split into **stability layers**:

```
┌──────────────────────────────────────────────────────────┐
│ LAYER 0: BEDROCK (1-hour cache)                          │
│ Identity, voice, hard constraints, relationship context  │
│ Changes: essentially never within a day                  │
│ ══════════════ CACHE BREAKPOINT 1 (1-hour) ════════════  │
├──────────────────────────────────────────────────────────┤
│ LAYER 1: SITUATIONAL (1-hour cache)                      │
│ Rules snapshot, entity registry, cost summary            │
│ Changes: a few times per day                             │
│ ══════════════ CACHE BREAKPOINT 2 (1-hour) ════════════  │
├──────────────────────────────────────────────────────────┤
│ LAYER 2: SESSION CONTEXT (5-min cache)                   │
│ Session memory, hot topics, recent processor outputs     │
│ Changes: per session / per escalation cycle              │
│ ══════════════ CACHE BREAKPOINT 3 (5-min)  ════════════  │
├──────────────────────────────────────────────────────────┤
│ HOT ZONE (never cached — owned by session)               │
│ Current event, thread context, engram hits, instructions │
│ Rewritten every call. Session-specific.                  │
├──────────────────────────────────────────────────────────┤
│ EPHEMERAL (never cached)                                 │
│ Raw tool results, drafts, intermediate reasoning         │
└──────────────────────────────────────────────────────────┘
```

Breakpoint 4 is held in reserve for future use or for the model's own tool-use turns in multi-turn Interactive conversations.

### Layer token budgets

```elixir
defmodule AgentRuntime.Cache.Layers do
  # Layer 0: Bedrock — identity core. 1-hour cache.
  # Same content for Haiku and Sonnet (Haiku gets compressed version
  # only if we can't meet the 4096 minimum — see "Haiku strategy" below)
  @layer0 %{
    personality_core: 2200,        # core personality tier
    personality_rich: 2200,        # rich personality tier (Sonnet/Interactive only)
    relationship: 800,             # relationship context
    voice_constraints: 500,        # voice, tone, hard behavioral rules
    total_haiku: 3500,             # compressed for Haiku's 4096 minimum
    total_sonnet: 5700             # full for Sonnet
  }

  # Layer 1: Situational — current operational state. 1-hour cache.
  # Changes when rules are written, entities groomed, or budget shifts significantly.
  @layer1 %{
    rules_snapshot: 800,           # active rules table, serialized
    entity_registry: 500,          # top-20 entities with context
    cost_summary: 300,             # last 24h spending, budget headroom
    total: 1600
  }

  # Layer 2: Session context — what's happening right now. 5-min cache.
  # Changes per escalation cycle. Multiple sessions can share if
  # the session context hasn't changed between them.
  @layer2 %{
    session_memory: 2000,          # compressed conversation/session history
    hot_topics: 500,               # detailed topic context
    human_preferences: 500,        # (Interactive only) current state, feedback
    project_context: 500,          # (Interactive only) what we're working on
    total_engaged: 2500,
    total_interactive: 3500
  }

  # Hot zone — never cached, session-owned
  @hot %{
    attentive: 2000,
    engaged: 4000,
    interactive: 6000
  }

  # Ephemeral — never cached
  @ephemeral %{
    attentive: 1000,
    engaged: 4000,
    interactive: 8000
  }
end
```

### Per-call totals

```
Attentive (Haiku):
  Layer 0: ~3500 (compressed identity, 1-hour cache)
  Layer 1: ~1600 (situational, 1-hour cache)    — total cached: ~5100
  Layer 2: not used (Haiku gets session context in hot zone instead)
  Hot: 2000 + Ephemeral: 1000
  Total: ~8100 tokens

Engaged (Sonnet):
  Layer 0: ~5700 (full identity, 1-hour cache)
  Layer 1: ~1600 (situational, 1-hour cache)    — total 1-hour cached: ~7300
  Layer 2: ~2500 (session context, 5-min cache) — total cached: ~9800
  Hot: 4000 + Ephemeral: 4000
  Total: ~17,800 tokens

Interactive (Sonnet):
  Layer 0: ~5700 (full identity, 1-hour cache)
  Layer 1: ~1600 (situational, 1-hour cache)    — total 1-hour cached: ~7300
  Layer 2: ~3500 (session + human, 5-min cache) — total cached: ~10,800
  Hot: 6000 + Ephemeral: 8000
  Total: ~24,800 tokens
```

---

## The Haiku Strategy

**Problem:** Haiku 4.5 has a 4,096-token minimum for caching. Layer 0 alone is 3,500 tokens for the compressed version — below minimum. Layer 1 is 1,600 tokens — also below minimum.

**Solution:** For Haiku, merge Layers 0+1 into a single cached block:

```
Haiku layout:
  [Layer 0 + Layer 1 combined: ~5100 tokens] ← BREAKPOINT 1 (1-hour)
  [Hot zone: 2000 tokens]
  [Ephemeral: 1000 tokens]
```

This means Haiku gets one breakpoint (combined identity + situational), not two. That's fine — Haiku calls are cheap and infrequent enough that the granularity doesn't matter. What matters is that the 5,100-token combined block is cached at the 1-hour tier and shared across all Haiku calls.

**For Sonnet** (2,048 minimum), both layers individually exceed the minimum, so we get the full two-breakpoint layout.

---

## Cache Layer Objects

The cache layers are first-class objects managed by `Cache.Manager`, independent of sessions:

```elixir
defmodule AgentRuntime.Cache.Layer do
  @type t :: %__MODULE__{
    id: String.t(),
    level: 0 | 1 | 2,
    ttl_strategy: :one_hour | :five_min,

    # content
    content: iodata(),
    content_hash: String.t(),        # SHA256 — detect staleness
    token_count: non_neg_integer(),

    # lifecycle
    created_at: DateTime.t(),
    last_hit_at: DateTime.t(),
    expires_at: DateTime.t(),        # resets on hit
    hit_count: non_neg_integer(),

    # divergence tracking
    source_hashes: %{                # hashes of the inputs that built this layer
      personality: String.t(),
      relationship: String.t(),
      rules: String.t(),
      entities: String.t(),
      # etc.
    },
    divergence_score: float()        # 0.0 = fresh, 1.0 = totally stale
  }
end
```

### Divergence scoring

When the underlying data changes (new rule written, entity tier changed, hot topic shifted), the layer's `divergence_score` increases. The question is always: **is the rewrite worth it?**

```elixir
defp should_rewrite_layer?(%Layer{} = layer) do
  remaining_ttl = DateTime.diff(layer.expires_at, DateTime.utc_now(), :second)
  expected_remaining_hits = estimate_hits(remaining_ttl, layer.hit_count, layer.created_at)

  # Cost of rewrite: tokens * write_multiplier
  rewrite_cost = layer.token_count * write_cost_per_token(layer.ttl_strategy)

  # Cost of staleness: divergence_score affects inference quality
  # (but a slightly stale rules snapshot doesn't meaningfully change behavior)
  staleness_cost = layer.divergence_score * quality_impact(layer.level)

  # Cost of doing nothing: read at 0.1x for remaining hits (already paid for)
  # So the question is: does the quality improvement justify the rewrite?
  staleness_cost * expected_remaining_hits > rewrite_cost
end
```

Most of the time, the answer is **don't rewrite**. A rules snapshot that's 30 minutes stale (one new rule added) has a divergence score of maybe 0.1. That's not worth a 2x write cost on a 5,700-token layer. Let it ride and rebuild fresh next time.

**When to rewrite:**
- Layer 0: almost never (personality doesn't change)
- Layer 1: when a significant rule change occurs AND the layer has many expected remaining hits (e.g., rules change + we're mid-Interactive session with 20 more turns expected)
- Layer 2: on session change (this is the designed refresh point)

---

## Session ↔ Cache Layer Relationship

Sessions don't own cache layers. They **reference** them:

```elixir
defmodule AgentRuntime.Session do
  # ... existing fields ...

  # Cache references (not ownership — layers outlive sessions)
  @type t :: %__MODULE__{
    # ...
    cache_layer_refs: %{
      layer0: String.t() | nil,      # Layer ID being used
      layer1: String.t() | nil,
      layer2: String.t() | nil
    },
    hot_zone: iodata()               # session OWNS this, rebuilt each call
  }
end
```

### Request assembly

When a session needs to make an API call:

```elixir
def assemble_request(%Session{} = session, ephemeral_content) do
  # Get or create cache layers
  layer0 = Cache.Manager.get_or_build_layer(0, :one_hour)
  layer1 = Cache.Manager.get_or_build_layer(1, :one_hour)
  layer2 = Cache.Manager.get_or_build_layer(2, :five_min, session)

  messages = [
    # Layer 0: identity (1-hour cache)
    %{role: "system", content: layer0.content,
      cache_control: %{type: "ephemeral", ttl: "1h"}},

    # Layer 1: situational (1-hour cache)
    %{role: "system", content: layer1.content,
      cache_control: %{type: "ephemeral", ttl: "1h"}},

    # Layer 2: session context (5-min cache)
    %{role: "system", content: layer2.content,
      cache_control: %{type: "ephemeral"}},  # default 5-min

    # Hot zone: session-specific, never cached
    %{role: "user", content: session.hot_zone},

    # Ephemeral: use and discard
    %{role: "user", content: ephemeral_content}
  ]

  # Touch all layers (extends TTL)
  Cache.Manager.touch(layer0.id)
  Cache.Manager.touch(layer1.id)
  Cache.Manager.touch(layer2.id)

  messages
end
```

### The parallel inference payoff

Two sessions within the same 5-minute window:

```
Session A (thread about identity):
  Layer 0: CACHE READ (0.1x)  ← written hours ago, still alive
  Layer 1: CACHE READ (0.1x)  ← written 20min ago, still alive
  Layer 2: CACHE WRITE (1.25x) ← new session context
  Hot zone: session A's event + context
  → Cost: 5700*0.1 + 1600*0.1 + 2500*1.25 + hot+ephemeral uncached

Session B (unrelated mention, 3 minutes later):
  Layer 0: CACHE READ (0.1x)  ← same as A, still alive
  Layer 1: CACHE READ (0.1x)  ← same as A, still alive
  Layer 2: CACHE READ (0.1x)  ← SAME session context, A touched it 3min ago!
  Hot zone: session B's event + context
  → Cost: 5700*0.1 + 1600*0.1 + 2500*0.1 + hot+ephemeral uncached
```

Session B's entire frozen zone is a cache read. The only uncached tokens are the hot zone and ephemeral — the stuff that's actually specific to this event. That's the optimal case: **you only pay for what's unique to this inference call.**

### The all-day cache scenario

With steady activity (at least one API call per hour), the 1-hour layers stay alive all day:

```
06:00  First Haiku call of the day
       Layer 0+1: CACHE WRITE (2x)     — $0.014 (5100 tokens at Haiku pricing)
       This is the only write cost for the entire day.

06:15  Second Haiku call
       Layer 0+1: CACHE READ (0.1x)    — $0.0001

...

23:45  Last Sonnet call of the day
       Layer 0: CACHE READ (0.1x)      — $0.0017 (Sonnet pricing)
       Layer 1: CACHE READ (0.1x)      — $0.0005

       Over 40 API calls today:
       1 write + 39 reads on Layer 0+1
       Write cost: $0.014
       Read cost: 39 × ~$0.001 = $0.039
       Total identity+situational caching cost: $0.053

       Without caching, same 40 calls:
       40 × 5100 tokens × $0.25/M (Haiku) + 40 × 7300 tokens × $3/M (Sonnet)
       = $0.051 + $0.876 = $0.927

       Savings: ~$0.87/day just from Layer 0+1 caching
       That's 94% reduction in frozen zone costs.
```

The math gets even better with Sonnet because its per-token cost is 12x higher. Every Sonnet cache read on Layer 0 saves ~$0.015 vs. uncached. Ten Sonnet calls/day = $0.15 saved on identity alone.

---

## Cache Manager Responsibilities

`Cache.Manager` is now the owner of all cache layer state. It's independent of sessions:

```elixir
defmodule AgentRuntime.Cache.Manager do
  @moduledoc """
  Manages cache layers independently of sessions.
  Layers have their own lifecycle — sessions reference them, don't own them.
  """

  # Get an existing valid layer or build a new one
  def get_or_build_layer(level, ttl_strategy, opts \\ [])

  # Touch a layer (extends TTL by the layer's TTL duration)
  def touch(layer_id)

  # Check if a layer should be rewritten (divergence scoring)
  def check_divergence(layer_id) :: {:fresh | :stale, float()}

  # Signal that a session is done — layer continues to live on its own TTL
  def session_released(layer_id)

  # List all live layers (for debugging/observability)
  def list_live_layers() :: [%Layer{}]
end
```

### Layer lifecycle

```
                    build (2x or 1.25x write)
                         │
                         ▼
                    ┌──────────┐
                    │   LIVE   │ ← TTL refreshes on every hit
                    │          │   multiple sessions can reference
                    └────┬─────┘
                         │
                    no hits for
                    TTL duration
                         │
                         ▼
                    ┌──────────┐
                    │ EXPIRING │ ← TaskScheduler notified, may dispatch
                    │          │   opportunistic work to keep it alive
                    └────┬─────┘
                         │
                    TTL expires
                         │
                         ▼
                    ┌──────────┐
                    │  DEAD    │ ← cleaned up, next request triggers rebuild
                    └──────────┘
```

The EXPIRING state is where TaskScheduler comes in: if a Layer 0+1 cache is about to expire and there's deferred work that could use it, dispatch that work to keep the cache alive. The deferred work is cheap (just hot+ephemeral tokens). Keeping a 7,300-token Sonnet Layer 0+1 alive for another hour costs one cheap inference call — potentially saving $0.015+ on the next real Sonnet call.

---

## Pre-Flight with Layers

Pre-flight now operates per-layer:

```elixir
defmodule AgentRuntime.Cache.PreFlight do
  def prepare_request(session, ephemeral) do
    tier = session.tier

    # Build or fetch each layer
    layer0 = build_or_fetch_layer(0, tier)
    layer1 = build_or_fetch_layer(1, tier)
    layer2 = build_or_fetch_layer(2, tier, session)

    # Token counting per zone
    counts = %{
      layer0: TokenCounter.estimate(layer0.content),
      layer1: TokenCounter.estimate(layer1.content),
      layer2: TokenCounter.estimate(layer2.content),
      hot: TokenCounter.estimate(session.hot_zone),
      ephemeral: TokenCounter.estimate(ephemeral)
    }

    total = Map.values(counts) |> Enum.sum()

    # Compression cascade (same priority as before):
    # 1. Truncate ephemeral
    # 2. Compress hot zone (ask Processor)
    # 3. Trim Layer 2 session_memory (least stable cached content)
    # NEVER compress Layer 0 or Layer 1

    if total > max_context_for_tier(tier) do
      compress_cascade(counts, session, ephemeral, tier)
    else
      assemble(layer0, layer1, layer2, session.hot_zone, ephemeral)
    end
  end
end
```

---

## Migration from v1

The conceptual change:

| v1 (original) | v2 (layered) |
|---|---|
| One frozen zone, one breakpoint | Three cached layers, three breakpoints |
| Session owns the frozen zone | Session references shared layers |
| Cache window = session lifetime | Cache layers = independent lifecycle, outlive sessions |
| 5-min or "kept alive" binary | 1-hour (bedrock, situational) + 5-min (session context) |
| One cache per session | Multiple sessions share one cache |

The implementation plan Phase 3A (Cache Zone Builder) absorbs this design. No other phases change — the rest of the system still sees "build frozen zone, build hot zone, make API call." The layering is internal to the cache subsystem.

---

## Design Properties

- **Layers are shared, hot zones are owned.** Any session can ride any layer whose content hash matches. Only the hot zone is session-specific. This is what makes parallel inference cheap.
- **Divergence is tolerable.** A slightly stale Layer 1 (one new rule since cache write) is almost always worth riding out. Only rewrite when the quality impact × remaining hits > rewrite cost.
- **1-hour cache is the default for stable content.** The 2x write cost seems expensive, but it's paid once. A single day of cache reads at 0.1x pays for the write dozens of times over.
- **Haiku and Sonnet share Layer 0 in spirit.** The content is the same (Haiku gets a compressed version). If both make calls within the same hour, both benefit from auto-refresh — Haiku's call keeps Sonnet's Layer 0 alive and vice versa, as long as the content hash matches. (In practice, Haiku's compressed Layer 0 has a different hash from Sonnet's full Layer 0, so they're separate cache entries. But they refresh independently and that's fine.)
- **The API is still one string.** All this layering happens at the prompt assembly level. The API sees one messages array with cache_control annotations. The model sees one coherent context. The layering is purely a cost optimization, invisible to the model.
