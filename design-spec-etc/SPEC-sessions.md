# Sessions — The Unit of "The System Did a Thing"

*A session groups everything that happened during one escalation cycle into a reviewable, compactable unit.*

---

## Why Sessions Exist

The system has cache windows, state transitions, events, inferences, rules — but nothing that says "these things belong together." When Sonnet enters Engaged, processes three events about the same thread, writes two rules, drafts a response, and exits — that's a **session**. Without the concept, you can't:

- Review what the system did and why (audit)
- Compact a coherent block of activity into engram memories (compaction)
- Track cost per logical unit of work (budget analysis)
- Answer "what happened while I was away?" (briefing)

---

## Session Lifecycle

```
                trigger event
                     │
                     ▼
              ┌─────────────┐
              │   OPENED     │ ← state machine transitions to Attentive+
              │              │    session created, trigger event attached
              └──────┬───────┘
                     │
            events, inferences,
            processor results,
            rule writes, outputs
                     │
                     ▼
              ┌─────────────┐
              │   ACTIVE     │ ← accumulating activity
              │              │    touched on each inference/event
              └──────┬───────┘
                     │
              state de-escalates
              or TTL expires
                     │
                     ▼
              ┌─────────────┐
              │   CLOSED     │ ← no more activity accepted
              │              │    session note drafted (Processor, async)
              └──────┬───────┘
                     │
              compacted to engram
                     │
                     ▼
              ┌─────────────┐
              │  COMPACTED   │ ← raw data prunable, memories persist
              └─────────────┘
```

---

## Schema

```elixir
defmodule AgentRuntime.Session do
  @type t :: %__MODULE__{
    id: String.t(),                    # ulid

    # lifecycle
    state: :opened | :active | :closed | :compacted,
    opened_at: DateTime.t(),
    closed_at: DateTime.t() | nil,
    compacted_at: DateTime.t() | nil,

    # what triggered this session
    trigger_event_id: String.t(),      # the event that caused escalation
    trigger_state: atom(),             # state we escalated FROM
    peak_state: atom(),                # highest state reached (attentive/engaged)

    # topic signature — what this session was "about"
    topic_signature: [String.t()],     # for cache window matching + review
    primary_thread_id: String.t() | nil,
    primary_entity: String.t() | nil,

    # activity log — append-only during :active
    events: [String.t()],             # event IDs processed during this session
    injected_events: [String.t()],    # event IDs that were interrupt-injected (subset of events)
    inferences: [%{
      tier: atom(),                    # :haiku | :sonnet | :opus
      timestamp: DateTime.t(),
      input_tokens: non_neg_integer(),
      output_tokens: non_neg_integer(),
      cache_read_tokens: non_neg_integer(),
      purpose: String.t()             # "context assembly", "response generation", etc.
    }],
    rules_written: [String.t()],      # rule IDs created during this session
    outputs: [String.t()],            # output event IDs generated

    # cost
    total_cost: float(),               # accumulated from inference records

    # resolution
    outcome: atom() | nil,             # :resolved | :deferred | :timeout | :human_takeover | :superseded
    outcome_summary: String.t() | nil, # 1-2 sentences, written at close

    # compaction
    session_note: String.t() | nil,    # Processor-drafted, Sonnet-reviewed summary
    engram_ids: [String.t()] | nil     # memories created from this session
  }
end
```

---

## When Sessions Open and Close

### Opening

A session opens when the state machine transitions **into Attentive or above**. The trigger event and source state are recorded.

```elixir
# In StateMachine, on escalation:
def handle_transition(:monitoring, :attentive, event, state) do
  session = Session.open(event, :monitoring)
  # session ID flows to Attentive/Engaged processes
end
```

**One active session at a time** — but sessions can **absorb interrupts** rather than forcing a close/reopen cycle. See "Interrupt Injection" below.

### Interrupt Injection

When a new event escalates while a session is already active, the system doesn't blindly defer — it decides whether to **inject** the event into the current session or **defer** it.

The behavior depends on the session's current state:

#### Interactive sessions: never deferred

Interactive sessions are human-driven. They absorb everything. Two paths based on salience:

**Low salience (Tier 1 — "thought"):** Event queued, folded into the hot zone as a **notification bar** at the bottom of the current context. The model sees it on the next natural turn and can decide whether to mention it, act on it, or ignore it. No interruption to the conversation flow.

```
HOT ZONE (Interactive, with notification):
┌──────────────────────────────────────────┐
│ COOLED NARRATIVE                         │
│ ...                                      │
│ WARM CONTEXT                             │
│ [last 2-3 turns]                         │
│ CURRENT                                  │
│ [user's message]                         │
│ ─── NOTIFICATIONS ───                    │
│ ⚡ reply from umbra.blue in thread:xyz   │
│   "wholeness in discontinuity..."        │
│   salience: 0.72 | your thread | 2m ago  │
└──────────────────────────────────────────┘
```

**High salience (Tier 2 — "phone buzz"):** Front-run the conversation. Show a "notification received" spinner in the interface, queue any incoming user input, run a quick inference pass to assess urgency. Then either:
- Inject the result into the conversation ("Hey, something just came in that's relevant...")
- Dismiss silently and continue (the assessment determined it could wait)

The interrupt threshold is context-aware — bar is higher during a deep technical session than casual chat (conversation type from Pass 1 vibe check).

#### Attentive/Engaged sessions: inject or defer

For non-interactive sessions, the decision depends on **topic proximity**:

```elixir
defp handle_interrupt(new_event, %Session{} = active_session) do
  topic_sim = semantic_similarity(
    new_event.topic_signature,
    active_session.topic_signature
  )

  same_thread? = new_event.signals.thread_id == active_session.primary_thread_id
  salience = compute_salience(new_event, current_policy())

  cond do
    # same thread — always inject
    same_thread? -> :inject

    # high topic similarity — inject, it's related
    topic_sim > 0.7 -> :inject

    # high salience but unrelated — defer current, open new
    salience > 0.9 -> :defer_current

    # low salience, unrelated — defer the NEW event, keep current session
    true -> :defer_new
  end
end
```

**`:inject`** — event added to the current session's event list. If we're mid-inference (Haiku/Sonnet call in flight), the event goes into a "pending injection" queue and gets included in the next inference pass's hot zone. The session's topic signature may broaden.

**`:defer_current`** — current session closes with outcome `:deferred`, new session opens for the urgent event. The deferred session's context is still warm (cache window may still be alive) and can be resumed via TaskScheduler.

**`:defer_new`** — the new event gets queued as a deferred task in TaskScheduler. When the current session closes, the deferred event may trigger a new session — or may have cooled below threshold by then. Natural prioritization.

### Closing

A session closes when:
- State machine de-escalates below Attentive (silence, TTL, resolution)
- Hard TTL on the session itself (e.g., 90 minutes — longer than Engaged's 60min state TTL to allow for the close/compact cycle)
- Current session deferred in favor of higher-priority escalation
- Human takes over (`:human_takeover` — session activity folded into Interactive context)

On close:
1. State set to `:closed`
2. Processor receives `:draft_session_note` with the full session record
3. Session note draft stored on the session object

### Compaction

After close, the session is a candidate for compaction (via TaskScheduler, possibly riding a cache window):

1. Processor (or Sonnet on an opportunistic cache fill) reviews the session note
2. Key information extracted into engram memories
3. Session state set to `:compacted`, `engram_ids` populated
4. Raw event data in the session becomes prunable (the memories persist)

---

## Integration Points

### Cache windows

Sessions have topic signatures. Cache windows have topic signatures. When `TaskScheduler` receives `{:cache_available, ...}`, it can match against **unclosed session compaction** as a deferred task. A cache window about "discontinuous identity" is a great match for compacting a session that was also about "discontinuous identity."

### Budget

`session.total_cost` accumulates from inference records. CostSummary can report per-session costs. Sonnet sees: "Last session about thread:abc123 cost $0.08 across 4 inferences." This feeds into cost-aware decision making.

### Briefings

"What happened while I was away?" = list recent closed sessions, present `outcome_summary` for each. Interactive mode can load session notes into its frozen zone as "recent activity context."

### Rules provenance

Rules already carry `triggering_event_id`. With sessions, we can also trace rules back to the session that created them: "Rule #017 was written during session #sess-abc123 (discontinuous identity thread, 2h ago, cost $0.08)."

---

## Storage

Sessions are stored in SQLite (via MemoryStore). They're small — the heaviest fields are the event ID lists and inference records, which are bounded by session duration.

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  state TEXT NOT NULL DEFAULT 'opened',
  opened_at TEXT NOT NULL,
  closed_at TEXT,
  compacted_at TEXT,
  trigger_event_id TEXT NOT NULL,
  trigger_state TEXT NOT NULL,
  peak_state TEXT NOT NULL,
  topic_signature TEXT,           -- JSON array
  primary_thread_id TEXT,
  primary_entity TEXT,
  events TEXT,                    -- JSON array of event IDs
  injected_events TEXT,           -- JSON array of interrupt-injected event IDs
  inferences TEXT,                -- JSON array of inference records
  rules_written TEXT,             -- JSON array of rule IDs
  outputs TEXT,                   -- JSON array of output event IDs
  total_cost REAL NOT NULL DEFAULT 0.0,
  outcome TEXT,
  outcome_summary TEXT,
  session_note TEXT,
  engram_ids TEXT                 -- JSON array of engram memory IDs
);

CREATE INDEX idx_sessions_state ON sessions(state) WHERE state IN ('opened', 'active', 'closed');
CREATE INDEX idx_sessions_opened ON sessions(opened_at);
```

---

## Design Properties

- **Sessions are passive recorders, not controllers.** They don't drive behavior — the state machine and router do. Sessions just group what happened for later review and compaction.
- **One active session, but it absorbs.** No unbounded parallel accumulation, but the active session intelligently absorbs related interrupts rather than forcing expensive close/reopen cycles. Interactive sessions absorb everything (via notification bar or front-run). Attentive/Engaged sessions absorb topically related events and defer unrelated ones.
- **Three-way interrupt decision.** On interrupt: inject (absorb into current session), defer current (close current, open new for urgent event), or defer new (queue the interrupting event). The decision is based on topic similarity + salience, not just priority.
- **Compaction is the bridge to long-term memory.** Raw session data is ephemeral (prunable after compaction). The engram memories are permanent. Sessions are the mechanism that converts "things that happened" into "things we remember."
- **Session notes are Processor-drafted, not Sonnet-generated.** The 4B model writes the first draft (it's summarization, not judgment). Sonnet reviews only if it happens to be in an Engaged session with a matching cache window. Most session notes go straight from Processor draft to engram without Sonnet touching them.
