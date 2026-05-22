# Design: Hybrid Deterministic Routing

*Draft 2026-04-03. Based on research in `stuff/research/routing-determinism-audit.md`.*

## Problem

Currently, `Myelin.InferenceRouter` performs two LLM inference calls per event to determine its routing outcome:
1. **Pass 1 (Vibe Check):** A fast LLM check (`Qwen3.5-2B`) on the first 100 characters to classify the message as `:quiet`, `:notable`, or `:urgent`.
2. **Pass 2 (Action Decision):** A second LLM check on the first 200 characters to decide the specific action (`:drop`, `:watch`, `:react`, `:escalate`).

This approach has significant flaws:
- **Wasted Effort:** Pass 2 is literally rubber-stamping hardcoded rules via an LLM. The prompt explicitly instructs the LLM to map sender tiers and salience scores to specific actions, acting as a slow, expensive, and potentially flaky `cond do` statement.
- **Blind Spots:** Pass 1 only sees text, missing crucial deterministic signals like numeric salience or sender tier. It can easily veto highly important messages simply because their text appears `:quiet`.
- **Cost & Latency:** We waste inference budget and add 100-300ms of latency per event on decisions that should be mathematically deterministic.

## Design

We will replace the two-pass LLM routing with a hybrid system: a deterministic fast-path paired with an LLM fallback for noisy text, and a purely deterministic Pass 2.

### Pass 1: Hybrid (Deterministic Fast-Path + LLM Fallback)

We will introduce a configurable `fast_path_threshold`.
- **Fast-Path:** Events with a salience score above the `fast_path_threshold`, OR events from Tier 1 senders, OR direct mentions (`event.kind == :mention`) will **skip Pass 1 entirely** and proceed directly to Pass 2. A small LLM reading 100 characters should not overrule high deterministic salience.
- **LLM Fallback:** Events in the ambiguous middle band (salience between `event_floor` and `fast_path_threshold`) will still run through the LLM vibe check. The LLM is highly useful here for filtering out noisy, low-signal text that accidentally accumulated enough points to pass the floor.
- **Dropped:** Events below the `event_floor` continue to be dropped immediately, as they are today.

### Pass 2: Fully Deterministic

The Pass 2 Action Decision LLM call will be completely removed and replaced with a pure Elixir routing function.

The function will map `(sender_tier, salience_band, event_kind)` directly to an `action` (`:drop`, `:watch`, `:react`, `:escalate`) and a human-readable `reason` string.

**Interaction with Rules System:**
Dynamic rules (from `MemoryStore.Rules`) apply *before* the routing table. They produce multipliers that modify the base salience score. The routing table reads this final, modified salience score to determine the `salience_band` and the resulting action.

#### Routing Function Sketch

The routing function will be pure and stateless:

```elixir
defmodule Myelin.Router.Deterministic do
  @moduledoc "Pure deterministic mapping for Pass 2 routing."

  alias Myelin.Event
  alias Myelin.Config.Routing, as: Config

  @doc """
  Determines the routing action.
  Input: Enriched %Event{} (with signals), current state machine state.
  Output: {:ok, action, reason}
  """
  def decide_action(%Event{} = event, config \\ default_config()) do
    tier = event.signals[:sender_tier] || 3
    salience = event.signals[:salience] || 0.0
    kind = event.kind
    
    salience_band = determine_band(salience, config.thresholds)

    # Build the signal bag: everything a rule can match against
    signals = %{
      "tier" => to_string(tier),
      "salience_band" => determine_band(salience, config.thresholds),
      "event_kind" => to_string(kind)
    }
    |> Map.merge(stringify_signals(event.signals))

    # Find the first matching rule in the configured action table
    match = Enum.find(config.actions, fn rule ->
      rule_matches?(rule, signals)
    end)

    if match do
      {:ok, match.action, match.reason || "matched rule"}
    else
      {:ok, :watch, "fallback: no explicit rule matched"}
    end
  end

  defp determine_band(salience, thresholds) do
    cond do
      salience >= thresholds.high_salience -> "high"
      salience >= thresholds.medium_salience -> "medium"
      true -> "low"
    end
  end

  defp rule_matches?(rule, signals) do
    # Every key in the rule (except action/reason) is a match clause.
    # Omitted keys are unconstrained (implicit wildcard).
    # "any" is an explicit wildcard. If a rule references a signal
    # not present on the event, the rule does not match.
    rule
    |> Map.drop([:action, :reason])
    |> Enum.all?(fn {key, value} ->
      value == "any" or Map.get(signals, to_string(key)) == value
    end)
  end
end
```

### Configuration: `config/routing.toml`

Following `Myelin.Config.Loader` standards (`DESIGN-config-standardization.md`), routing policies will be defined in TOML. The configuration is parsed once and passed to the routing function.

```toml
# config/routing.toml
name = "default-routing"
type = "routing_policy"

[thresholds]
event_floor = 0.2
fast_path_threshold = 0.8
high_salience = 0.7
medium_salience = 0.4

# Action table: evaluated top-to-bottom, first match wins.
# Every key except "action" and "reason" is a match clause against event signals.
# "any" is a wildcard. Unrecognized signal keys = rule doesn't match (logged at load).

[[actions]]
tier = "1"
action = "escalate"
reason = "tier 1 sender"

[[actions]]
salience_band = "high"
event_kind = "mention"
action = "react"
reason = "high salience mention"

[[actions]]
salience_band = "high"
action = "react"
reason = "high salience"

[[actions]]
salience_band = "medium"
action = "watch"
reason = "medium salience"

[[actions]]
salience_band = "low"
action = "drop"
reason = "low salience"

# Example: future enrichment signal — rule matches only if signal is present
# [[actions]]
# salience_band = "medium"
# platform = "telegram"
# action = "react"
# reason = "medium salience on primary platform"
```

## Migration Path

Myelin is pre-deployment. No baseline behavior to preserve — rip and replace.

1. **Wave 1 (Build):**
   - Create `Myelin.Router.Deterministic` (pure module, extensible rule matching).
   - Create TOML config loader for `config/routing.toml`.
   - Create `config/routing.toml` with default rules.
   - Tests for the deterministic router and config loading.

2. **Wave 2 (Wire + Cleanup):**
   - Rewire `InferenceRouter` to use deterministic Pass 2 and hybrid Pass 1.
   - Remove dead Pass 2 LLM prompt templates (`render_action/5`).
   - Remove the `action_decision_flow/4` logic from `inference_router.ex`.
   - Update existing router tests to reflect the new flow.

## What Doesn't Change

- **Salience Scoring:** The enrichment pipeline (`Myelin.Salience`) remains untouched.
- **Rules System:** Dynamic multipliers from `MemoryStore.Rules` are untouched. They feed *into* the deterministic routing table.
- **StateMachine:** Still receives the exact same routing outcomes (`:drop`, `:watch`, `:react`, `:escalate`) and reasons.
- **State-Specific Handlers:** `:attentive` and `:engaged` loop handlers remain untouched.

## Design Decisions

- **Configurable via TOML, not runtime-mutable:** An autonomous agent should not modify its own core routing thresholds. The operator sets the policy; the agent follows it.
- **Pure Function:** Routing is a stateless computation over signals, making it extremely fast, testable, and robust (no GenServer bottleneck).
- **Extensible Rule Matching:** Rules can match on ANY signal present on the event, not just tier/salience_band/event_kind. Reserved keys are `action` and `reason` — everything else is a match clause. `"any"` is a wildcard. If a rule references a signal not present on the event, the rule doesn't match. This means enriching events with new signals automatically makes them routable without code changes — just add a clause to `routing.toml`.
- **Reason Strings:** Every deterministic decision must return a human-readable reason to maintain observability and explainability in logs, replacing the LLM's generated "reason" field.
- **Terminology:** `mention` is social-media-specific for what we actually mean (direct address / explicit invocation). Future rename planned — a `sed` job, not a refactor.

## Future Considerations

- **Budget Reallocation:** The inference budget freed by eliminating the Pass 2 LLM and fast-pathing Pass 1 can be redirected to deeper response generation for `:react` events.
- **Granular Overrides:** The routing table schema could eventually support `interface` or `conversation_id` fields to allow per-platform or per-thread deterministic overrides.
- **Rules Subsumption:** If the `MemoryStore.Rules` system evolves significantly, it could potentially subsume the TOML routing table entirely (since thresholds are essentially just fixed multiplier boundaries).
