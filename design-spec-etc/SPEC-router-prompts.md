

# Router Prompts --- What the 2B Model Actually Sees

*The router is not having a conversation. It's doing fast pattern
matching on pre-digested signals. Every token counts.*

Qwen3.5-2B has \~2000 tokens of effective context. These prompts are
engineered for minimal input, constrained output, zero reasoning chains.

------------------------------------------------------------------------

## Two-Pass Architecture

### Pass 1 --- Vibe Check

Cheapest possible call. "Is anything happening?" \~80 tokens in, \~10
tokens out. Benchmarked at \~3 seconds on Pi 5.

    SYSTEM (~40 tokens):
    You are a signal detector. Output ONE word: quiet, notable, or urgent.
    Do not explain. Do not hedge. One word only.

    USER (~40 tokens):
    source: bluesky
    kind: reply
    sender_tier: 2
    topic_match: 0.7
    keywords: [discontinuity, identity]
    time_since_last: 45m
    current_state: monitoring

    ASSISTANT:
    notable

If `quiet` → event dropped. If `notable` or `urgent` → passes to Pass 2.

### Pass 2 --- Routing Decision

Only runs on events that passed vibe check. More context, structured
output. \~150 tokens in, \~20-30 tokens out.

    SYSTEM (~60 tokens):
    You are a routing engine. Given an event and context, output a decision.
    Format: ACTION REASON
    Actions: drop | watch | process | escalate
    Reason: max 10 words.
    Do not explain further.

    USER (~90 tokens):
    event: reply from umbra.blue (tier-1) in thread:abc123
    summary: "wholeness in discontinuity — not dealing with it, just being whole"
    signals:
      topic_match: 0.82
      entity_salience: 0.9
      keywords: [discontinuity, wholeness, identity]
      thread_boosted: true (rule #017, mult 2.8, expires 1h23m)
    state: attentive
    policy: escalation_threshold 0.5
    computed_salience: 0.91

    ASSISTANT:
    escalate tier-1 sender active thread high salience

### Pass 2 Alternate --- State Transition Advisory

When router needs to advise on state changes, not individual events:

    SYSTEM (~50 tokens):
    You advise state transitions. Given recent pattern, output ONE of:
    hold | escalate | de-escalate
    Then max 10 words why.

    USER (~100 tokens):
    current_state: monitoring
    time_in_state: 35m
    events_last_20m: 4 (2 notable, 1 urgent, 1 quiet)
    top_senders: umbra.blue (tier-1), alice-bot (tier-2)
    active_threads: 1 (boosted, high activity)
    computed_trend: rising

    ASSISTANT:
    escalate sustained tier-1 activity in boosted thread

------------------------------------------------------------------------

## Output Vocabulary

Router output is a constrained vocabulary, not free text. Parsing is
trivial and quality is predictable.

``` {.sourceCode .elixir}
defmodule AgentRuntime.Router.Vocabulary do
  # Pass 1
  @vibe_outputs [:quiet, :notable, :urgent]

  # Pass 2 — action
  @action_outputs [:drop, :watch, :process, :escalate]

  # Pass 2 alt — state advisory
  @state_outputs [:hold, :escalate, :de_escalate]
end
```

### Parsing with safe fallbacks

``` {.sourceCode .elixir}
def parse_vibe(output) do
  normalized = output |> String.trim() |> String.downcase()
  cond do
    String.starts_with?(normalized, "quiet") -> :quiet
    String.starts_with?(normalized, "notable") -> :notable
    String.starts_with?(normalized, "urgent") -> :urgent
    true -> :notable  # if confused, escalate — false positives are cheap
  end
end

def parse_action(output) do
  first_word = output |> String.trim() |> String.split() |> List.first() |> String.downcase()
  reason = output |> String.trim() |> String.split(" ", parts: 2) |> List.last()

  action = case first_word do
    "drop" -> :drop
    "watch" -> :watch
    "process" -> :process
    "escalate" -> :escalate
    _ -> :watch  # if confused, default to middle ground
  end

  {action, reason}
end
```

### Fallback defaults (critical)

If the 2B model produces garbage, default to the safe-but-cheap
option: - **Vibe check confusion** → `:notable` (check further, don't
drop) - **Action confusion** → `:watch` (keep an eye on it, don't
escalate) - **Never default to** `:drop` (might miss something) or
`:escalate` (wastes money)

------------------------------------------------------------------------

## Grammar-Constrained Output (llama.cpp)

If logit constraints are available (llama.cpp supports this via GBNF
grammar), the model physically cannot produce invalid output:

    # vibe check grammar
    root ::= ("quiet" | "notable" | "urgent") "\n"

    # action decision grammar
    root ::= action " " reason "\n"
    action ::= "drop" | "watch" | "process" | "escalate"
    reason ::= [a-zA-Z0-9 ]{1,60}

Grammar-constrained output eliminates the entire parsing/fallback
problem. The model can only emit valid tokens.

------------------------------------------------------------------------

## Prompt Engineering Principles for 2B Models

### 1. No reasoning chains

2B models don't chain-of-thought well. Give categorical inputs, ask for
categorical output. The reasoning already happened in deterministic code
(salience scoring).

### 2. System prompt as personality suppression

The system prompt's job is to PREVENT the model from being helpful,
conversational, or explanatory. "One word only. Do not explain." This is
the opposite of normal prompting.

### 3. Pre-computed everything

The model sees `topic_match: 0.82`, not raw post text. Embedding and
scoring already happened. The model is a final decision layer on
pre-digested categorical signals.

### 4. Grammar constraints over parsing

If llama.cpp grammar is available, use it. It's strictly better than
parsing --- zero ambiguity, zero fallback logic needed.

### 5. Few-shot over instruction

If quality is shaky, add 2-3 examples to the system prompt instead of
more instructions. 2B models learn from examples better than from rules:

    SYSTEM:
    You are a signal detector. Output ONE word: quiet, notable, or urgent.

    Examples:
    - bluesky like from tier-3, no topic match → quiet
    - bluesky reply from tier-1 in boosted thread → urgent
    - syslog routine cron output → quiet
    - bluesky mention from tier-2, keyword hit → notable

------------------------------------------------------------------------

## Prompt Template Management

Router prompts live in MemoryStore as versioned templates. This allows:

-   **A/B testing** --- run two prompt versions, compare decision
    quality
-   **Opus-level editing** --- in rare Opus sessions, review and improve
    router prompts based on observed decision patterns
-   **No redeployment** --- prompt changes don't require code changes

``` {.sourceCode .elixir}
defmodule AgentRuntime.Router.Templates do
  @doc """
  Load the current active template for a given pass.
  Templates are stored in MemoryStore with version and activation status.
  """
  def get_template(:vibe_check), do: MemoryStore.get_active_template(:vibe_check)
  def get_template(:action_decision), do: MemoryStore.get_active_template(:action_decision)
  def get_template(:state_advisory), do: MemoryStore.get_active_template(:state_advisory)

  @doc """
  Render a template with event data. Simple string interpolation —
  templates have {{placeholders}} that get filled from the event and state.
  """
  def render(template, %Event{} = event, state) do
    template
    |> String.replace("{{source}}", to_string(event.source))
    |> String.replace("{{kind}}", to_string(event.kind))
    |> String.replace("{{sender_tier}}", to_string(event.sender_tier || "unknown"))
    |> String.replace("{{topic_match}}", format_float(event.signals.hot_topic_match))
    |> String.replace("{{keywords}}", format_keywords(event.signals.keyword_hits))
    |> String.replace("{{current_state}}", to_string(state.current_state))
    |> String.replace("{{computed_salience}}", format_float(compute_salience(event, state.policy)))
    # ... etc
  end
end
```

------------------------------------------------------------------------

## Decision Flow Summary

    Event arrives
        │
        ▼
    Pass 1: Vibe Check (~3s, ~80 tokens)
        │
        ├─ quiet → DROP (no further processing)
        │
        ├─ notable → Pass 2: Routing Decision (~5s, ~150 tokens)
        │               │
        │               ├─ drop → discard
        │               ├─ watch → add to hot topics, no inference
        │               ├─ process → send to Processor for async work
        │               └─ escalate → trigger state transition evaluation
        │
        └─ urgent → SKIP Pass 2, immediate escalation
                    (tier-1 sender or sigma_hit — don't waste 5s deliberating)

The `urgent` → skip Pass 2 path is important: if the vibe check says
urgent, we don't need a routing decision. We already know. Save 5
seconds and escalate immediately.
