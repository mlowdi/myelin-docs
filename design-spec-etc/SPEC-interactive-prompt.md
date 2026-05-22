# Interactive Session Prompt System

*Each API call is a new story. The Interactive session's job is to make every token earn its place — and to make the model's output do multiple jobs simultaneously.*

---

## The Core Insight

A naive interactive session grows linearly: each turn appends the full exchange to the context. After 20 turns you have 20× the context of turn 1, most of it redundant. The model re-reads everything it already knows every single call.

The alternative: **each turn is a fresh narrative** built from the minimum viable signal to produce a high-quality response. The model is always "just arrived" with excellent briefing notes — not "has been sitting here for 3 hours reading the transcript."

The structured output schema is the mechanism that makes this work: the model produces the response AND maintains the session's working memory as a side effect of responding.

---

## Structured Output Schema

Every Interactive session turn uses this schema:

```json
{
  "type": "object",
  "properties": {
    "response_text": {
      "type": "string",
      "description": "The conversational response to the user. This is what appears in the chat window."
    },
    "narration": {
      "type": "string",
      "description": "1-3 sentences in first person describing this exchange for your own session memory. Will be appended to the session narrative. Be specific about what was asked, what you said, and any decisions or shifts that matter for continuity."
    },
    "rewrite_narrative": {
      "type": "string",
      "description": "Optional. If the session narrative has drifted, become unwieldy, or needs restructuring, provide a full replacement here. Leave null to just append the narration. Use this when the existing narrative no longer accurately represents the conversation or has grown too long to be useful."
    },
    "tool_calls": {
      "type": "array",
      "description": "Async tool calls to fire. Results arrive in future turns. See tool system spec.",
      "items": {
        "type": "object",
        "properties": {
          "op": { "type": "string" },
          "args": { "type": "object" },
          "ref": { "type": "string" },
          "failure_mode": {
            "type": "string",
            "enum": ["silent", "next_turn", "inject"],
            "description": "How to handle failure. silent: log only. next_turn: note in next natural turn. inject: interrupt immediately."
          }
        },
        "required": ["op", "args", "ref", "failure_mode"]
      }
    },
    "awaiting": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Refs of async tool calls still pending. System injects these into the hot zone each turn until resolved."
    }
  },
  "required": ["response_text", "narration"]
}
```

**What each field does:**

| Field | Goes to | Cost per turn |
|---|---|---|
| `response_text` | User's chat window | — |
| `narration` | Appended to session narrative in hot zone | ~30-50 tokens |
| `rewrite_narrative` | Replaces session narrative (optional) | ~100-200 tokens |
| `tool_calls` | Parsed by system, dispatched async | — |
| `awaiting` | Injected into next turn's hot zone | ~10 tokens per pending ref |

A complex 400-word response compacts to 2 narration sentences. The hot zone grows by ~40 tokens per turn regardless of response length. That's the entire compaction mechanism — no separate Processor call needed until the narrative gets too long to be useful (at which point the model rewrites it itself).

---

## Session Narrative

The session narrative is the model's **working memory of the conversation** — distinct from the conversation transcript.

### Structure

```
SESSION NARRATIVE
─────────────────
Turn 1: User asked about cloud-first strategy; I gave a mechanism-first explanation
  tailored to ops context, emphasized CapEx→OpEx shift and shared responsibility.
Turn 2: User pushed back on security claims; I acknowledged shared-responsibility
  caveat, agreed hyperscaler baseline is high but not sufficient alone.
Turn 3: Shifted to Myelin — user wants to understand structured output for
  Interactive sessions. Explained "each API call is a new story."
[... continues ...]
```

### Token growth

Without narrative compaction: ~200 tokens/turn (full exchange transcript)
With narrative compaction: ~35 tokens/turn (2 sentence narration)

After 20 turns:
- Naive: ~4,000 tokens of growing transcript
- With narration: ~700 tokens of dense session memory

### Caching the narrative

The session narrative lives in the hot zone — normally uncached. But if the narrative has grown to >500 tokens, it's a signal that the session has been going a while and will likely continue. At that point, cache the narrative block in Layer 2 (5-min):

```elixir
defp should_cache_narrative?(narrative) do
  token_count = TokenCounter.estimate(narrative)
  token_count > 500  # grown enough to be worth caching
  # if it's this long, session is active, 5-min cache is worth it
end
```

A cached narrative saves ~150 tokens × 0.9 = ~135 tokens per call at 0.1x read cost. Over a long session this adds up.

### Rewrite trigger

The model decides when to rewrite. Signals it might notice:
- The narrative has become a list of facts rather than a coherent thread
- Earlier turns are no longer relevant to current direction
- The session pivoted significantly and old context is now misleading
- Narrative length is approaching the point of diminishing returns

The system can also *suggest* a rewrite by including `[narrative is N tokens, consider rewriting]` in the hot zone framing when it gets long.

---

## Hot Zone Construction (Interactive)

```
HOT ZONE:
┌──────────────────────────────────────────────────────┐
│ CURRENT CONVERSATIONS REGISTER (from Layer 2 cache)  │
│ You are in an Interactive session.                   │
│ Other active conversations: [list]                   │
│                                                      │
│ SESSION NARRATIVE                                    │
│ [self-maintained, grows ~35 tokens/turn]             │
│ [possibly cached if >500 tokens]                     │
│                                                      │
│ PENDING TOOL CALLS                                   │
│ Awaiting: search-abc123 (web_search, fired 2 turns   │
│   ago), enrich-xyz456 (enrich_memories, just fired)  │
│                                                      │
│ RECENT TURNS (last 2-3)                              │
│ [literal transcript of recent exchanges]             │
│ [anchor for continuity — narrative tells the arc,    │
│  recent turns tell the last thing said]              │
│                                                      │
│ CURRENT INPUT                                        │
│ [user's message / injected tool result]              │
└──────────────────────────────────────────────────────┘
```

The narrative tells the arc. Recent turns anchor the thread. Together they replace a full growing transcript while costing a fraction of the tokens.

---

## System Prompt Framing

The system prompt (in Layer 2) tells the model how to use the schema:

```
You are in an Interactive session with Martin. You have a structured output schema.

response_text: Your conversational reply. This is what Martin sees.

narration: 1-3 sentences describing this exchange for your own memory. First person,
specific. "Martin asked X; I explained Y; we agreed on Z." This gets appended to
your session narrative so future-you knows what happened here.

rewrite_narrative: Use this when your session narrative has drifted or grown unwieldy.
Replace it with a clean, dense summary of the conversation so far. Leave null to
just append your narration.

tool_calls: Async operations to kick off. Results arrive in future turns — you won't
have them in this response. Declare failure_mode for each: "silent" (just log),
"next_turn" (note it when you naturally check in), or "inject" (interrupt me
immediately if this fails, it's critical).

You can see what you're waiting on in PENDING TOOL CALLS above. Results arrive
either in the next natural turn or as injected turns for urgent/failed items.

The window metaphor: response_text is what you type into the chat window Martin sees.
Everything else is what's happening in your workspace around you.
```

---

## Design Properties

- **Self-maintaining context.** The model manages its own working memory as a side effect of responding. No separate compaction trigger, no threshold monitoring, no Processor call needed for routine sessions.

- **One call, multiple outputs.** Response + narration + async tool dispatch happen in one API call. The model is doing real work every turn.

- **Optimistic async.** Tool results are reported on success in the next natural turn. Failures interrupt only when declared critical. The model's default assumption is that the world is working.

- **The window metaphor.** The model has a chat window (response_text) and a workspace (everything else). This framing makes the dual-output schema intuitive rather than confusing.

- **Graceful degradation.** If the model forgets to include narration, the session still works — just without self-maintenance. The worst case is falling back to a slightly longer hot zone, not a broken session.
