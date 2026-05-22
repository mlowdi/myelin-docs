

# State Machine --- Threshold Policies and Guard Conditions

*States are threshold policies, not activity labels. Each defines what
cost/value ratio justifies escalation.*

------------------------------------------------------------------------

## State Diagram

    ┌─────────┐     salience      ┌────────────┐    escalation    ┌───────────┐
    │ Dormant │ ──────cluster────▶ │ Monitoring │ ───predicted───▶ │ Attentive │
    └─────────┘                    └────────────┘                  └───────────┘
         ▲                              ▲                            │       │
         │          silence/TTL         │      resolved/false alarm  │       │
         └──────────────────────────────┴────────────────────────────┘       │
                                                                            │
                                                                 low conf   │
                                                                            ▼
                                                                   ┌──────────┐
                                        human connects             │ Engaged  │
                                        Any ──────────▶ Interactive└──────────┘

------------------------------------------------------------------------

## Threshold Policies

Each state defines a `Policy` struct that the router uses for all
decisions within that state:

``` {.sourceCode .elixir}
defmodule AgentRuntime.StateMachine.Policy do
  @type t :: %__MODULE__{
    event_floor: float(),         # minimum salience score to even process
    escalation_threshold: float(),# score above which router considers escalating
    max_backend: atom(),          # highest tier available in this state
    cache_strategy: atom(),       # :none | :short | :long
    decay_rate: float(),          # how fast we cool back toward dormant
    interrupt_floor: float()      # for Interactive: threshold to interrupt conversation
  }
end
```

### Policy Values

  ----------------------------------------------------------------------------------------------------------
  State             event_floor   escalation_threshold   max_backend    cache      decay   interrupt_floor
  ----------------- ------------- ---------------------- -------------- ---------- ------- -----------------
  **Dormant**       0.7           0.9                    `:router`      `:none`    0.0     1.0

  **Monitoring**    0.3           0.7                    `:processor`   `:none`    0.05    1.0

  **Attentive**     0.1           0.5                    `:haiku`       `:short`   0.1     1.0

  **Engaged**       0.05          0.8                    `:sonnet`      `:long`    0.15    1.0

  **Interactive**   0.1           0.5                    `:sonnet`      `:long`    0.0     0.8
  ----------------------------------------------------------------------------------------------------------

Key observations: - **Dormant** is nearly blind --- event_floor 0.7
means most things are invisible, escalation threshold 0.9 means only
extraordinary events wake us - **Engaged** has a HIGH escalation
threshold (0.8) because going beyond Sonnet means Opus, which is rare
and expensive - **Interactive** doesn't decay --- human is driving. But
it has an active interrupt_floor for background events - **decay_rate**
increases with cost --- Engaged cools fastest because it's the most
expensive state

------------------------------------------------------------------------

## Salience Scoring

Salience is computed deterministically from pre-stamped Event signals.
NOT by the router model. This is fast math, not inference.

``` {.sourceCode .elixir}
defp compute_salience(%Event{} = event, %Policy{} = policy) do
  base = 0.0

  # sender tier is the biggest lever
  base = base + case event.sender_tier do
    1 -> 0.6    # tier-1: almost always worth looking at
    2 -> 0.3    # tier-2: notable
    3 -> 0.1    # tier-3: stranger
    nil -> 0.05 # unknown sender
  end

  # hot topic match — if this event is about something we're tracking
  base = base + (event.signals.hot_topic_match || 0.0) * 0.3

  # entity salience — the person's general importance score
  base = base + (event.signals.entity_salience || 0.0) * 0.2

  # keyword hits from rules table
  base = base + min(length(event.signals.keyword_hits) * 0.15, 0.4)

  # kind multiplier — some event types are inherently more salient
  multiplier = case event.kind do
    :mention -> 1.5      # someone talked TO us
    :reply -> 1.3        # in a thread we're in
    :message -> 1.4      # direct message
    :sigma_hit -> 1.8    # security event (security application)
    :follow -> 0.8       # nice but not urgent
    :like -> 0.3         # least urgent
    :cache_available -> 0.0  # internal scheduling, not salience-driven
    _ -> 1.0
  end

  min(base * multiplier, 1.0)
end
```

### Scoring breakdown

-   **Sender tier** (0.05--0.6): Biggest single factor. Tier-1 sender
    alone gets 0.6 base --- with a `:mention` multiplier that's 0.9,
    which clears Dormant's escalation threshold
-   **Hot topic match** (0.0--0.3): Semantic similarity to things we're
    actively tracking
-   **Entity salience** (0.0--0.2): General importance of the person
    (accumulated over time)
-   **Keyword hits** (0.0--0.4, capped): Rules table keywords --- Sonnet
    can add these dynamically
-   **Kind multiplier** (0.3x--1.8x): Structural importance of event
    type

------------------------------------------------------------------------

## Guard Conditions

### Dormant → Monitoring

*"Salience cluster" --- not one event, a PATTERN*

``` {.sourceCode .elixir}
defp guard(:dormant, :monitoring, event, state) do
  salience = compute_salience(event, @policies.dormant)

  cond do
    # single extraordinary event
    salience >= 0.9 -> :transition

    # tier-1 sender, period
    event.sender_tier == 1 -> :transition

    # cluster: 3+ events above floor in 20 minutes
    recent_above_floor = count_recent_events(state, minutes: 20, min_salience: 0.7)
    recent_above_floor >= 3 -> :transition

    true -> :stay
  end
end
```

### Monitoring → Attentive

*"Router predicts escalation likely; worth assembling context"*

``` {.sourceCode .elixir}
defp guard(:monitoring, :attentive, event, state) do
  salience = compute_salience(event, @policies.monitoring)

  cond do
    # salience exceeds escalation threshold
    salience >= 0.7 -> :transition

    # active thread with us as participant + new reply
    event.kind in [:reply, :mention] and
      is_our_thread?(event.signals.thread_id, state) -> :transition

    # sustained activity: 5+ events above monitoring floor in 10 minutes
    recent = count_recent_events(state, minutes: 10, min_salience: 0.3)
    recent >= 5 -> :transition

    true -> :stay
  end
end
```

### Attentive → Engaged

*"Haiku assembled brief, confidence low, Sonnet judgment needed"*

This transition is DIFFERENT --- triggered by Haiku's output, not raw
event signals. This is the designed escalation point where inference
output feeds back into the state machine.

``` {.sourceCode .elixir}
defp guard(:attentive, :engaged, _event, state) do
  case state.haiku_result do
    %{confidence: conf} when conf < 0.6 -> :transition
    %{recommends_escalation: true} -> :transition
    _ -> :stay
  end
end
```

### Attentive → Monitoring (de-escalation)

*"Haiku resolves: false alarm, thread dead"*

``` {.sourceCode .elixir}
defp guard(:attentive, :monitoring, _event, state) do
  cond do
    state.haiku_result && state.haiku_result.resolved -> :transition
    events_since_escalation(state) == 0 and
      minutes_in_state(state) > 5 -> :transition
    true -> :stay
  end
end
```

### Engaged → Dormant

*TTL expiry or sustained silence*

``` {.sourceCode .elixir}
defp guard(:engaged, :dormant, _event, state) do
  cond do
    minutes_in_state(state) > 60 -> :transition           # hard TTL
    events_since_last_inference(state) == 0 and
      minutes_since_last_inference(state) > 15 -> :transition  # silence
    true -> :stay
  end
end
```

### Any → Interactive

``` {.sourceCode .elixir}
defp guard(_any, :interactive, event, _state) do
  if event.kind == :human_connect, do: :transition, else: :stay
end
```

------------------------------------------------------------------------

## Key Design Properties

### Router model doesn't make transition decisions

Guard conditions are deterministic code operating on pre-computed
signals. The router's actual job is narrower: "given this event and this
policy, what should I do with it?" *within* a state. The state machine
moves between states based on math.

**One exception:** Attentive → Engaged, where Haiku's confidence score
drives the transition. This is the designed escalation point --- the
first time inference output feeds back into the state machine.

### Cluster detection, not single-event triggers

Dormant → Monitoring requires a *pattern*: 3 events above floor in 20
minutes, or a tier-1 sender, or a single extraordinary event. This
prevents noisy event sources from keeping the system perpetually warm.

### Asymmetric escalation/de-escalation

Escalation requires evidence. De-escalation requires *absence* of
evidence (silence, TTL expiry). This is intentional --- it's cheaper to
stay warm for a few extra minutes than to re-escalate.

### All numbers are tunable

Every threshold, every weight, every TTL is a named constant. The rules
table (see SPEC-rules-table.md) allows Sonnet to adjust some of these
dynamically. The rest are tuned through operational experience.
