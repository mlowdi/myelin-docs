# Temporal Context in LLM Prompts

## Principle

**Surface temporal relations, not raw timestamps.** LLMs are language models, not math models — they cannot subtract ISO timestamps to derive meaning. Every prompt we build (routing, context assembly, vibe check, action decision) must express time as *what it means*, not *when it was*.

## Why

We're building **story to react to**. A timestamp is inert data — "this happened at this point." The model doesn't know nor care. It needs the *temporal relation*: how long ago, how much time between events, what the rhythm of a conversation looks like.

This applies at every level of the inference chain:
- **Router (Qwen 2B):** "last message in thread: 2m ago" vs. a raw timestamp. The router can parse "2m ago" into urgency. It cannot subtract timestamps.
- **Attentive (Haiku):** "Monday 2026-03-16T14:30:00+01:00 (1 day, 15h 12m since last message)" gives both anchor and delta.
- **Engaged (Sonnet):** Full conversation timeline with readable gaps between turns.

## Format

```
<weekday> <ISO timestamp in local timezone> (<readable delta> since last event)
```

Examples:
- `Monday 2026-03-16T14:30:00+01:00 (2m since last message)`
- `Sunday 2026-03-15T23:18:00+01:00 (1 day, 15h 12m since last message)`
- `last message in thread: 2m ago` (compact form for routing prompts)

### Delta formatting rules

| Range | Format | Example |
|-------|--------|---------|
| < 1 minute | `Xs ago` | `45s ago` |
| < 1 hour | `Xm ago` | `12m ago` |
| < 24 hours | `Xh Ym ago` | `3h 15m ago` |
| >= 24 hours | `X day(s), Yh Zm ago` | `1 day, 15h 12m ago` |

### Token economy

For routing prompts (where every token counts), use the compact form: `last msg: 2m ago`. For context assembly where we have budget, use the full form with weekday + ISO + delta.

## Implementation Notes

- The delta should be computed at prompt assembly time, not stored
- `DateTime.diff/2` gives seconds; format into the tiers above
- A shared helper (e.g., `Myelin.TimeFormat.relative_delta/2`) should handle formatting so all prompt builders are consistent
- Local timezone comes from config (we're single-timezone for now)
