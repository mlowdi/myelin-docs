# Design Principle: Context as Scratchpad, Not Transcript

*The context window isn't a story of what happened. It's a scratchpad for writing the next part of the story.*

---

## The Principle

In a naive agent architecture, the context window is a transcript: events happen, they get appended, the model reads the whole history and decides what to do. The context IS the system's state.

In this architecture, the context window is a **manufactured briefing**. The system's state lives in deterministic structures (salience scores, guard conditions, rules table, entity registry, hot topics register, session records). The context window is assembled fresh for each inference call from those structures, showing the model only what it needs to make the decision at hand.

The context is not *what happened*. It's *what matters right now*.

---

## What This Means in Practice

### ~80% deterministic, ~20% inference

| Deterministic (tunable, auditable, replayable) | Inference (expensive, powerful, rare) |
|---|---|
| Event ingestion + signal stamping | Router vibe check (2B, ~3s) |
| Salience scoring (math on pre-stamped signals) | Router Pass 2 decision (2B, ~5s) |
| Rules table application (multiplicative, with decay) | Processor operations (4B, async) |
| Guard conditions (code on computed values) | Haiku assessment + research (API) |
| State machine transitions | Sonnet reasoning + rule writing (API) |
| Cache layer management | |
| Budget tracking + enforcement | |
| Task scheduling + priority routing | |
| Session lifecycle | |

Inference only fires on pre-digested material. The model never sees raw firehose — it sees a briefing manufactured by deterministic code and cheaper models.

### The model writes code that runs between its own invocations

Sonnet's rule-writing capability means it can reprogram the deterministic layer:
- Boost a thread → salience scoring math changes
- Suppress an entity → guard conditions behave differently
- Pin a state → decay function overridden
- Conservation mode → threshold policy modified

This isn't the model "learning" — it's the model writing configuration for a deterministic system. The configuration has TTLs, decay, hard caps. It's auditable: every rule has provenance (who wrote it, why, what triggered it).

### Consequences for the context window

Because the context is a manufactured scratchpad:

1. **It can be rebuilt from scratch at any time.** The system's state is in MemoryStore, not in the conversation history. If a cache expires, rebuild the context from current state. No information is lost.

2. **Multiple sessions can share the same base context.** The identity layer (Layer 0) is the same regardless of which event we're processing. Only the hot zone changes per session.

3. **The hot zone is a register, not a log.** It's rewritten each call, not appended to. In Interactive mode, the narrative compaction system keeps it bounded: each turn generates a summary that replaces the raw transcript. The scratchpad stays clean.

4. **Ephemeral content is truly ephemeral.** Tool results, intermediate reasoning, draft outputs — used once, signal extracted, discarded. They never become part of the system's persistent state.

5. **The context budget is a design constraint, not a limitation.** We're not fighting the context window — we're designing INTO it. The frozen/hot/ephemeral zone structure is a deliberate architecture that makes the context window work harder per token.

---

## The Operational Payoff

Because 80% is deterministic:

- **Replay:** Given the same event + state snapshot, the system produces the same routing decision. You can debug any decision after the fact.
- **A/B testing:** Change a threshold, replay a day's events, compare decisions. No model calls needed.
- **Audit trail:** Every escalation has a complete causal chain: event signals → salience score → rule multipliers → guard condition → transition → context assembly → inference → output.
- **Self-tuning:** Sonnet adjusts the deterministic layer through rules. The Processor analyzes rule effectiveness. The system tightens its own routing over time through operational reflection, not training.
- **Cost predictability:** Deterministic code is free. Only inference costs money. And with cached identity layers, even inference is predictably cheap.

The system is transparent by construction. Not because we added logging (though we do log everything) — because the architecture separates "what the system does" (deterministic, inspectable) from "what the model thinks" (inference, expensive, rare).
