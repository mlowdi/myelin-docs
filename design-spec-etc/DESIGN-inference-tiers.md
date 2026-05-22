# Design: Inference Capability Tiers

*Draft 2026-03-29. Connects to: DESIGN-backend-capabilities.md, DESIGN-pluggable-memory.md (metabolism grooming needs cheap evaluation)*

## Problem

The backend router selects models by capability + cost. But cost-sorting means the cheapest model always wins within a capability set. There's no way to express "I need reasoning, and it needs to be *good* reasoning" vs "I need reasoning, and I need it cheap."

This blocks:
- **Metabolism/grooming** (Wave 4): needs cheap evaluation, not SOTA reasoning
- **Model diversity**: can't add a 32B Mistral on OpenRouter without it stealing all `:processing` tasks from Sonnet (because it's cheaper) even when Sonnet-quality is needed
- **Interactive sessions**: should get the best available model, not the cheapest

## Current state

Capabilities are flat atoms: `:routing`, `:reasoning`, `:processing`, `:chat`, `:tool_use`, etc. Callers request a set of capabilities. The router filters backends that have all requested capabilities, then sorts by `{priority, cost_per_output_token, load, latency}`.

No quality signal. No tier distinction. A 4B local model and Sonnet can both claim `:processing`, and the local model wins on cost every time.

## Proposed: Tier as a first-class dimension

### The tier axis

Add a `tier` field to backend configs. Four tiers:

| Tier | Meaning | Examples |
|------|---------|----------|
| `:edge` | Must run on constrained hardware (RPi5, no GPU). Latency-critical, cost-zero. | Local Qwen 0.5B, llama.cpp on ARM |
| `:efficient` | Good task-following, text transformation. Cost-optimized. Not expected to reason deeply. | 32B Mistral/Qwen on OpenRouter, local Qwen 4B via Cortex |
| `:capable` | SOTA task-following. Excellent instruction adherence, text transformation, evaluation. Fast. | Haiku, Gemini Flash |
| `:frontier` | SOTA reasoning. Complex decisions, planning, multi-step analysis. | Sonnet, Opus, Gemini Pro |

### How callers request tiers

Add `min_tier` to `Inference.Request`:

```elixir
%Inference.Request{
  capability: :reasoning,
  min_tier: :capable,    # "at least Haiku-class"
  locality: :any,
  urgency: :normal
}
```

`min_tier` is a *floor*, not an exact match. Requesting `:capable` means "capable or better" — the router may return a `:frontier` backend if no `:capable` backend is available or if the `:frontier` backend happens to be cheaper (unlikely, but possible).

Tier ordering: `:edge` < `:efficient` < `:capable` < `:frontier`

If `min_tier` is not specified, defaults to `:efficient` (don't accidentally send stuff to the edge model unless explicitly requested).

### Backend TOML change

```toml
# config/backends/openrouter-mistral.toml
name = "openrouter-mistral-32b"
model = "mistralai/mistral-small-3.2-24b-instruct-2506"
endpoint = "https://openrouter.ai/api/v1"
calling_convention = "openai"
locality = "remote"
tier = "efficient"
capabilities = ["processing", "summarization", "feature_extraction", "chat"]

[cost]
input = 0.10
output = 0.30

# config/backends/anthropic-haiku.toml
name = "anthropic-haiku"
model = "claude-haiku-4-5-20251001"
endpoint = "https://api.anthropic.com"
calling_convention = "anthropic"
locality = "remote"
tier = "capable"
capabilities = ["processing", "chat", "structured_output", "summarization", "feature_extraction", "tool_use"]

[cost]
input = 0.80
output = 4.00

# config/backends/anthropic-sonnet.toml
name = "anthropic-sonnet"
model = "claude-sonnet-4-6"
endpoint = "https://api.anthropic.com"
calling_convention = "anthropic"
locality = "remote"
tier = "frontier"
capabilities = ["processing", "chat", "reasoning", "structured_output", "tool_use", "agent", "interactive"]

[cost]
input = 3.00
output = 15.00
```

### Router changes

`BackendPool.route_task/2` gains tier filtering:

```elixir
def route_task(capabilities, opts \\ []) do
  min_tier = Keyword.get(opts, :min_tier, :efficient)

  backends
  |> filter_by_status(urgency)
  |> filter_by_capabilities(capabilities)
  |> filter_by_locality(locality)
  |> filter_by_min_tier(min_tier)          # NEW
  |> sort_by({:priority, :cost, :load, :latency})
  |> List.first()
end

defp filter_by_min_tier(backends, min_tier) do
  tier_rank = %{edge: 0, efficient: 1, capable: 2, frontier: 3}
  min_rank = Map.fetch!(tier_rank, min_tier)
  Enum.filter(backends, fn b -> Map.get(tier_rank, b.tier, 0) >= min_rank end)
end
```

After tier filtering, cost-sorting does the right thing: within `:capable`+, Haiku is cheaper than Sonnet, so Haiku wins — which is correct. A 32B Mistral at `:efficient` tier won't steal work from Haiku when `:capable` is requested.

### Routing strategy

Additionally, allow callers to specify a sorting strategy:

```elixir
# Default: prefer cheapest (current behavior)
route_task([:processing], min_tier: :efficient, strategy: :prefer_cost)

# For interactive sessions: prefer lowest latency
route_task([:interactive, :reasoning], min_tier: :frontier, strategy: :prefer_speed)

# For background batch work: prefer cheapest, allow degraded backends
route_task([:processing], min_tier: :efficient, strategy: :prefer_cost, urgency: :low)
```

Strategies:
- `:prefer_cost` — sort by `{cost, latency, load}` (default, current behavior)
- `:prefer_speed` — sort by `{latency, load, cost}`
- `:prefer_quality` — sort by `{-tier_rank, latency, cost}` (highest tier first, then fastest)

## Caller audit — what changes

Based on the capability audit, here's what each callsite should request:

| Callsite | Current | Proposed |
|----------|---------|----------|
| `inference_router.ex:172` (vibe check) | `:routing` | `:routing, min_tier: :edge` — this is the fast-path classification |
| `inference_router.ex:215` (action decision) | `:routing` | `:routing, min_tier: :edge` |
| `engaged.ex:109` (engaged response) | `:reasoning` | `:reasoning, min_tier: :frontier` — this is real thinking |
| `attentive.ex:59` (attentive response) | `:reasoning` | `:reasoning, min_tier: :capable` — monitoring, not deep analysis |
| `research_thread/runner.ex:125` | `:reasoning` | `:reasoning, min_tier: :frontier` — research needs real reasoning |
| `processor.ex:263` (batch processing) | `:batch` or `:processing` | `:processing, min_tier: :efficient` — bulk work, cheap is fine |
| `interactive.ex:145` (interactive) | `:interactive` | `:interactive, min_tier: :frontier, strategy: :prefer_speed` |
| `web_fetch.ex:216` (summarization) | `:processing` | `:processing, min_tier: :efficient` — text extraction |
| **Metabolism grooming** (Wave 4) | N/A | `:processing, min_tier: :capable` — evaluation needs good judgment, not reasoning |
| **Thread summarization** | N/A | `:summarization, min_tier: :efficient` — perfect for mid-tier OpenRouter models |

## Migration path

### Wave A: Add tier to backend infrastructure (no behavior change)
1. Add `tier` field to `Backend.Config` struct (default `:capable` for backwards compat)
2. Parse `tier` from TOML configs
3. Add `tier` to runtime `Backend` struct
4. Update all existing TOML configs with explicit tier values
5. Add `min_tier` to `Inference.Request` (default `:efficient`)
6. Add tier filtering to `BackendPool.route_task`

### Wave B: Add routing strategies
7. Add `strategy` option to `route_task`
8. Implement `:prefer_cost`, `:prefer_speed`, `:prefer_quality` sort orders

### Wave C: Update callers
9. Update each callsite with appropriate `min_tier` and `strategy` values (see table above)
10. Add OpenRouter backend TOML configs for mid-tier models

### Wave D: Wave 4 (metabolism) can now proceed
11. Grooming evaluator requests `:processing, min_tier: :capable` — gets Haiku or equivalent
12. `random_recall` wiring, threshold detection, evaluation batching

## What this enables

- **Model diversity without quality regression**: Add cheap OpenRouter models freely — they won't steal work from Haiku/Sonnet because callers specify `min_tier`
- **Cost optimization**: Thread summarization, feature extraction, and text transformation route to $0.10/M input models instead of $3/M Sonnet
- **Interactive quality guarantee**: `min_tier: :frontier, strategy: :prefer_speed` ensures the user always gets the best model, fast
- **Metabolism unlocked**: Grooming doesn't need to hard-code "use Haiku" — it requests `:capable` tier and the router picks the best option
- **Future-proof**: When a new model drops that's better than Haiku but cheaper than Sonnet, just assign it `tier: capable` and it participates in routing automatically

## Resolved questions

- **Tier taxonomy is universal.** Four tiers are enough for 80%+ of deployment scenarios. If someone wants to change the tier architecture, they can learn Elixir or get an AI to do it.
- **Quality scoring is impractical.** Instead, future work: a "backend quality callback" where admin + reasoning models can flag a backend as producing poor output ("this data sucks, bench this backend for a while"). Mostly a frontier model problem — some days models just act extra dumb.
- **Provider-specific concerns (OpenRouter, Mistral, Cleura, etc.)** are deferred to the inference backend implementation layer, not the routing tier system.

## Future possibilities unlocked by this design

- **Warm cache routing**: If a `:frontier` backend has a warm cache, routing a summarization job to it may be net-beneficial even though it's "overqualified." The `min_tier` floor + cost-sorting naturally enables this — warm cache makes the effective cost competitive.
- **Sovereign load management**: A `:sovereign :capable` backend that timeshares with other systems can be managed by extending the routing strategy (e.g., `:prefer_available`) or adjusting how load factors into the sort. The tier architecture doesn't need to change for this.
