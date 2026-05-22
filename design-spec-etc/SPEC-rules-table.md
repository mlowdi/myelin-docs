

# Rules Table --- Dynamic Routing Reprogramming

*Where Sonnet gets to rewrite the router's brain. With guardrails.*

The core idea: Sonnet, in the Engaged state with full reasoning, can
look at what just happened and write rules that modify how the salience
scorer and guard conditions behave going forward. This is the system's
immune system learning from exposure --- not the system rewriting its
own DNA.

------------------------------------------------------------------------

## Rule Schema

``` {.sourceCode .elixir}
defmodule AgentRuntime.Rules do
  @type rule_kind ::
    :boost_thread          # increase salience for events in a specific thread
    | :boost_entity        # increase salience for events from a specific sender
    | :boost_keyword       # add a keyword to watch for across all events
    | :suppress_thread     # decrease salience for a noisy thread
    | :suppress_entity     # decrease salience for a spammy sender
    | :pin_state           # prevent decay below a certain state for a duration
    | :escalation_override # force next matching event to escalate (one-shot)
    | :custom_prompt       # specialized routing prompt for matching events

  @type t :: %__MODULE__{
    id: String.t(),                    # ulid
    kind: rule_kind(),

    # targeting — what does this rule apply to?
    target: %{
      thread_id: String.t() | nil,
      entity_handle: String.t() | nil,
      keyword: String.t() | nil,
      source: atom() | nil,            # :bluesky | :syslog | nil (all sources)
      kind_filter: atom() | nil        # only apply to specific event kinds
    },

    # effect
    multiplier: float(),               # 0.1 = suppress, 3.0 = boost

    # custom prompt fields (only for :custom_prompt rules)
    custom_prompt: String.t() | nil,   # ≤100 tokens, specialized routing prompt
    prompt_tier: atom() | nil,         # :router | :haiku — which model runs it

    # lifecycle
    created_at: DateTime.t(),
    expires_at: DateTime.t(),          # hard expiry, no exceptions
    ttl_seconds: non_neg_integer(),    # original TTL (for logging/debugging)
    decay_function: atom(),            # :none | :linear | :exponential

    # provenance — WHY does this rule exist?
    created_by: atom(),                # :sonnet | :opus | :manual
    triggering_event_id: String.t() | nil,
    reason: String.t(),                # human-readable, for debugging and audit

    # state
    active: boolean(),
    fired_count: non_neg_integer()     # how many times this rule affected a score
  }
end
```

------------------------------------------------------------------------

## Hard Caps (Compile-Time Constants)

These are NOT configurable by Sonnet. They're the guardrails on the
guardrails.

``` {.sourceCode .elixir}
defmodule AgentRuntime.Rules.Limits do
  # maximum simultaneous active rules
  @max_active_rules 20

  # maximum TTL for any single rule
  @max_ttl_hours 24

  # multiplier bounds — can't go infinitely hot or completely silent
  @max_boost_multiplier 5.0
  @min_suppress_multiplier 0.1       # can suppress to 10%, never to zero

  # per-kind limits
  @max_boost_threads 5               # can't watch everything
  @max_boost_entities 5
  @max_boost_keywords 10
  @max_suppress_threads 10           # more generous — suppression is cheap
  @max_suppress_entities 10
  @max_pin_state_rules 2             # very limited — this is expensive
  @max_escalation_overrides 3        # one-shot rules, very limited
  @max_custom_prompts 3              # specialized routing prompts, tight limit
  @max_custom_prompt_tokens 100      # keep them short — this runs on 2B or Haiku

  # rate limits
  @max_rules_per_hour 10
  @max_rules_per_engaged_session 5   # per cache window
end
```

### Why these limits

-   **20 max active rules** --- enough to track several conversations
    and a few entities, not enough to turn the router into a christmas
    tree
-   **24h max TTL** --- nothing should persist indefinitely through
    rules. If it matters for longer, it should be an entity tier change
    or a hot topic in MemoryStore
-   **Can't suppress to zero** --- the 0.1 floor means even suppressed
    events can still trigger if other signals are strong enough. No
    total blindness
-   **2 max pin_state rules** --- pinning is expensive (prevents natural
    cooling), so it's heavily limited
-   **5 rules per engaged session** --- prevents Sonnet from going on a
    rule-writing spree during a single cache window

------------------------------------------------------------------------

## Rule Operations (Sonnet's API)

These are the operations available to Sonnet as structured output / tool
calls:

### `boost_thread(thread_id, opts)`

After posting into a thread, boost it so we catch replies. Most common
rule. Natural social media behavior.

Defaults: multiplier 3.0, duration 2h, linear decay.

### `boost_entity(handle, opts)`

Someone interesting appeared. Watch for them. Used after a good
conversation or discovering a new voice.

Defaults: multiplier 2.0, duration 6h, linear decay.

### `boost_keyword(keyword, opts)`

A topic is developing. Watch for it across all sources.

Defaults: multiplier 2.5, duration 4h, linear decay.

### `suppress_thread(thread_id, opts)`

Thread is noisy and uninteresting. Stop waking up for it.

Defaults: multiplier 0.2, duration 4h, no decay (suppression doesn't
decay --- it expires).

### `suppress_entity(handle, opts)`

Sender is spammy or uninteresting right now.

Defaults: multiplier 0.3, duration 6h, no decay.

### `pin_state(min_state, opts)`

Keep the system warm. Used when Sonnet knows more events are coming and
doesn't want to pay re-escalation cost.

Heavily limited: max 2 active, max 1h duration.

### `escalation_override(target_filter, opts)`

One-shot: next event matching this condition escalates immediately. Used
when Sonnet expects a specific reply or follow-up.

Auto-deactivates after firing once. Max 3 active. Max 1h TTL.

------------------------------------------------------------------------

## Decay Functions

Rules don't just expire --- they can fade. The multiplier decays toward
1.0 (neutral) over the rule's lifetime.

### `:none`

Multiplier stays constant until expiry. Used for suppression (binary:
either suppressed or not) and one-shot rules.

### `:linear`

Multiplier moves linearly toward 1.0. A 3.0 boost with 2h TTL at the 1h
mark has effective multiplier 2.0.

``` {.sourceCode .elixir}
defp apply_decay(%Rule{decay_function: :linear} = rule) do
  progress = elapsed_seconds / rule.ttl_seconds  # 0.0 → 1.0
  rule.multiplier + (1.0 - rule.multiplier) * progress
end
```

### `:exponential`

Multiplier decays exponentially with half-life at TTL/3. Front-loads the
effect --- strong initially, fading fast.

``` {.sourceCode .elixir}
defp apply_decay(%Rule{decay_function: :exponential} = rule) do
  half_life = rule.ttl_seconds / 3
  decay = :math.exp(-0.693 * elapsed_seconds / half_life)
  1.0 + (rule.multiplier - 1.0) * decay
end
```

------------------------------------------------------------------------

## Integration: Salience Scoring

Rules are applied AFTER base salience scoring, as a multiplicative
layer:

``` {.sourceCode .elixir}
defp apply_rules(base_salience, %Event{} = event) do
  active_rules = MemoryStore.get_active_rules()

  multiplier = active_rules
    |> Enum.filter(&rule_matches?(&1, event))
    |> Enum.reduce(1.0, fn rule, acc ->
      decayed = apply_decay(rule)
      MemoryStore.increment_fired_count(rule.id)
      acc * decayed
    end)
    |> max(0.1)    # floor
    |> min(5.0)    # ceiling

  base_salience * multiplier
end
```

### Rule matching

A rule matches an event if ALL non-nil target fields match:

``` {.sourceCode .elixir}
defp rule_matches?(%Rule{} = rule, %Event{} = event) do
  (rule.target.thread_id == nil or rule.target.thread_id == event.signals.thread_id) and
  (rule.target.entity_handle == nil or rule.target.entity_handle == event.sender) and
  (rule.target.keyword == nil or rule.target.keyword in extract_words(event.summary)) and
  (rule.target.source == nil or rule.target.source == event.source) and
  (rule.target.kind_filter == nil or rule.target.kind_filter == event.kind)
end
```

This means rules can be broad (boost all events from an entity) or
narrow (boost mentions of a keyword in Bluesky replies only). The
targeting is conjunctive --- all specified fields must match.

------------------------------------------------------------------------

## Provenance and Audit

Every rule carries full provenance:

-   **`created_by`** --- which model tier wrote this rule (`:sonnet`,
    `:opus`, `:manual`)
-   **`triggering_event_id`** --- the specific event that caused Sonnet
    to write this rule
-   **`reason`** --- human-readable string Sonnet writes explaining why

Example audit trail:

    Rule #017 | boost_entity("umbra.blue") | mult: 2.0 | decay: linear | TTL: 6h
    Created by: :sonnet | Event: #evt-abc123
    Reason: "umbra posted about discontinuous identity — active research topic,
             want to catch follow-up discussion"
    Fired: 3 times | Expires: 2026-03-13T04:30:00Z

This means you can inspect the rules table at any time and understand
exactly why the system is paying attention to what it's paying attention
to. Full chain of reasoning, not a black box.

------------------------------------------------------------------------

## Maintenance

MemoryStore runs periodic cleanup:

-   **Expiry sweep** --- every 60s, deactivate rules past `expires_at`
-   **Fired count logging** --- rules that never fire are interesting
    (Sonnet's prediction was wrong)
-   **Limit enforcement** --- if a new rule would exceed per-kind
    limits, oldest rule of that kind is deactivated first (LRU)
-   **Rate limit check** --- `validate_and_insert!()` rejects if hourly
    or per-session limit is hit

------------------------------------------------------------------------

## SQLite Schema

``` {.sourceCode .sql}
CREATE TABLE rules (
  id TEXT PRIMARY KEY,
  kind TEXT NOT NULL,
  target_thread_id TEXT,
  target_entity_handle TEXT,
  target_keyword TEXT,
  target_source TEXT,
  target_kind_filter TEXT,
  multiplier REAL NOT NULL,
  created_at TEXT NOT NULL,
  expires_at TEXT NOT NULL,
  ttl_seconds INTEGER NOT NULL,
  decay_function TEXT NOT NULL DEFAULT 'none',
  created_by TEXT NOT NULL,
  triggering_event_id TEXT,
  reason TEXT NOT NULL,
  active INTEGER NOT NULL DEFAULT 1,
  fired_count INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_rules_active ON rules(active) WHERE active = 1;
CREATE INDEX idx_rules_expires ON rules(expires_at) WHERE active = 1;
CREATE INDEX idx_rules_kind ON rules(kind) WHERE active = 1;
```

------------------------------------------------------------------------

## Custom Prompt Rules

A third category of rule: Sonnet can write **specialized routing
prompts** that run instead of the standard Pass 2 for matching events.
The system writing its own reflexes.

### Example

    Rule #023 | kind: :custom_prompt | TTL: 6h | decay: none
    target: {source: :syslog, keyword: "auth_failure"}
    prompt_tier: :haiku
    prompt: "Three or more auth failures from the same source IP
             in this batch? If yes: urgent. If no: quiet."
    reason: "saw discussion about credential stuffing,
             want to catch related log patterns while it's fresh"

For the next 6 hours, syslog events matching `auth_failure` get a
specialized prompt that knows what to look for instead of the generic
vibe check. When the rule expires, the system forgets it was worried
about credential stuffing --- unless the pattern keeps appearing, in
which case Sonnet re-writes the rule or promotes it to a permanent
routing template.

### Integration with router flow

``` {.sourceCode .elixir}
defp check_custom_prompts(event, active_rules) do
  matching_prompts = active_rules
    |> Enum.filter(&(&1.kind == :custom_prompt))
    |> Enum.filter(&rule_matches?(&1, event))

  case matching_prompts do
    [] -> :no_custom  # proceed with normal Pass 2
    [prompt_rule | _] ->
      result = case prompt_rule.prompt_tier do
        :router -> run_on_router(prompt_rule.custom_prompt, event)
        :haiku -> run_on_haiku(prompt_rule.custom_prompt, event)
      end
      {:custom, result, prompt_rule.id}
  end
end
```

### Constraints

-   Max 3 active custom prompts
-   Max 100 tokens per prompt
-   Output must use the same constrained vocabulary
    (quiet/notable/urgent or drop/watch/process/escalate)
-   Can run on `:router` (2B, free) or `:haiku` (API, cheap but not
    free)
-   Same provenance logging as all rules --- who wrote it, why, what
    triggered it

### Lifecycle

Custom prompts follow the same lifecycle as other rules: Sonnet writes
them during Engaged sessions, they have TTLs, they fire and are counted,
and `analyze_rules` reviews their effectiveness. A custom prompt that
fires often and leads to good escalation decisions is a candidate for
promotion to a permanent routing template (via Opus review).

------------------------------------------------------------------------

## Cost Awareness and Nerd-Snipe Protection

### The autoimmune spiral problem

An interesting thread can create a self-sustaining loop: boost thread →
events pass filter → escalate → Sonnet engages → Sonnet boosts thread
harder → more events pass → infinite Sonnet session. Hard caps prevent
the worst case, but the best defense is **Sonnet seeing its own spending
and self-correcting.**

### Cost context in Engaged sessions

Every time Sonnet enters an Engaged session, the frozen zone includes a
**cost summary**:

    COST CONTEXT (last 24h):
      API calls: 47 (12 Haiku, 34 Sonnet, 1 Opus)
      Tokens: 142k input (89k cached), 31k output
      Estimated cost: $0.38
      Budget: $0.50/day
      Headroom: $0.12

      Top cost drivers:
        thread:abc123 (discontinuous identity) — 18 calls, $0.14
        thread:def456 (elixir architecture) — 9 calls, $0.08
        scheduled maintenance — 6 calls, $0.04

      Active boosts consuming budget:
        boost_thread abc123 (2.8x, expires 1h23m) — 12 fires
        boost_entity umbra.blue (2.0x, expires 4h) — 8 fires
        custom_prompt auth_failure (haiku, expires 5h) — 3 fires

Sonnet can see: "I've spent \$0.14 on this one thread today. That's 37%
of my daily budget on one conversation. Maybe I should let the boost
decay instead of renewing it."

### Self-limiting operations

Sonnet has explicit operations for cost management:

``` {.sourceCode .elixir}
# "I'm spending too much on this, let it cool"
def reduce_exposure(thread_id) do
  # deactivate boosts for this thread
  # let decay happen naturally
  # optionally add a suppress rule to actively cool it
end

# "This topic is interesting but not urgent, defer it"
def defer_thread(thread_id, resume_condition) do
  # suppress the thread now
  # add a one-shot escalation_override for when the condition is met
  # e.g. "if umbra posts again in this thread, wake me up"
end

# "I've been too active today, lower all thresholds"
def enter_conservation_mode(duration_hours) do
  # temporarily raise event_floor across all states
  # only tier-1 senders and sigma hits get through
  # auto-expires after duration
end
```

### Budget as a hard constraint

The daily budget (\$0.50 default, configurable) is enforced at the
InferenceRouter level. If budget is exhausted:

-   Engaged → disabled, system drops to Attentive max
-   Attentive → continues (Haiku is cheap, \~\$0.01 per call)
-   Router → always available (local model, free)
-   Interactive → always available (human present = override)

Budget resets at a configured time (e.g., 10:00 local). The system
naturally gets quieter as it approaches budget exhaustion ---
self-limiting through economics, not rules.
