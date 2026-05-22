

# Processor --- Heavy Async Operations

*The system's digestive tract. Breaks down raw material into forms the
expensive models can use efficiently.*

The Processor runs on the 4B local model (OpenStack VM, Cleura). All
operations are fire-and-forget from the caller's perspective: send a
message, continue working, receive the result back as a message later.
No blocking. Pure OTP GenServer with async replies.

------------------------------------------------------------------------

## Atom Vocabulary

### Memory Operations

#### `:compress_thread`

Compress a thread into a dense summary suitable for the hot zone. Input:
full thread (fetched via payload_ref). Output: ≤200 token summary.

Used when Attentive is assembling context for Sonnet.

    {:compress_thread, thread_id, [%Event{}, ...]}
    → {:compressed, thread_id, summary_text, token_count, key_entities}

#### `:summarize_memories`

Summarize a batch of engram memories into a briefing paragraph. Used
when Attentive pulls semantic matches and needs them digestible.

    {:summarize_memories, request_id, [%{text: ..., similarity: ...}, ...]}
    → {:summarized_memories, request_id, briefing_text, token_count}

#### `:extract_entities`

Extract entities, topics, and sentiment from raw text. Used at ingestion
for richer signal stamping, or post-conversation for entity registry
updates.

    {:extract_entities, event_id, raw_text}
    → {:entities, event_id, [%{name: ..., type: ..., sentiment: ...}]}

#### `:rewrite_hot_zone`

Rewrite the hot zone register. Takes current hot zone + new information,
produces a fresh hot zone under the token budget. This is the "register,
not log" principle --- rewritten, not appended.

    {:rewrite_hot_zone, current_hot_zone, new_facts, token_budget}
    → {:hot_zone_updated, new_hot_zone_text, token_count}

------------------------------------------------------------------------

### Routing Support

#### `:deep_classify`

Deeper classification when the 2B router is uncertain. Pass 2.5 ---
between router and Haiku escalation. Cheaper than Haiku, smarter than
2B. Can resolve ambiguous events without burning API tokens.

    {:deep_classify, event_id, %Event{}, context_snippet}
    → {:classification, event_id, %{salience: float, category: atom, confidence: float}}

#### `:explain_routing`

Generate a routing explanation for audit/debugging. "Why did the system
escalate this?" --- answered by 4B looking at the event, applied rules,
and state at decision time.

    {:explain_routing, event_id, %Event{}, applied_rules, state_snapshot}
    → {:explanation, event_id, human_readable_text}

------------------------------------------------------------------------

### Content Preparation

#### `:draft_response`

Draft a response for Sonnet to review/edit rather than write from
scratch. Saves Sonnet tokens by giving it something to react to. Used
for routine social interactions where the shape is predictable.

    {:draft_response, thread_id, context_summary, tone_hints}
    → {:draft, thread_id, draft_text, confidence}

If confidence is high enough, Haiku can approve the draft directly
without Sonnet involvement. This is where cost stays at \$0.20/day.

#### `:digest_content`

Pre-digest a long document, article, or thread dump into a structured
brief: key claims, entities, relevance assessment. Used when an event
references external content.

    {:digest_content, request_id, raw_content, relevance_context}
    → {:digested, request_id, %{summary: ..., claims: [...], entities: [...], relevance: float}}

------------------------------------------------------------------------

### Maintenance (Scheduled)

#### `:groom_entities`

Periodic entity registry grooming. Reviews recent interactions, suggests
tier changes, updates salience scores. Runs on a schedule (every \~6h
via TaskScheduler), not per-event.

    {:groom_entities, recent_interactions_window}
    → {:entity_updates, [%{handle: ..., new_tier: ..., new_salience: ..., reason: ...}]}

#### `:analyze_rules`

Analyze rules table effectiveness. Which rules fired often? Which never
fired? What patterns suggest new rules? Feeds into Sonnet's next Engaged
session as context.

    {:analyze_rules, time_window}
    → {:rules_analysis, %{effective: [...], wasted: [...], suggested: [...]}}

#### `:compact_to_memory`

Compact old session context into engram-ready memories. The "move from
RAM to disk" operation in the cache hierarchy.

    {:compact_to_memory, session_context, importance_threshold}
    → {:compacted, [%{text: ..., importance: float}]}

------------------------------------------------------------------------

## Concrete Flow Example

Someone in a boosted thread replies to us:

    1. EventIngester.Bluesky emits %Event{kind: :reply, sender_tier: 2,
       signals: %{thread_id: "xyz", hot_topic_match: 0.7}}

    2. AgentStateMachine in Monitoring. Guard fires → Attentive.
       (boosted thread + reply + tier-2 = salience 0.78, above threshold)

    3. InferenceRouter under Attentive policy: need context assembly.

    4. Router sends THREE async messages to Processor:
       - {:compress_thread, "xyz", [previous events in thread]}
       - {:summarize_memories, req1, [engram hits for thread topic]}
       - {:extract_entities, evt_id, full_post_text}

    5. Router continues processing other events. Does NOT wait.

    6. Processor returns arrive (5-15s later, 4B model on VM):
       - {:compressed, "xyz", "thread about discontinuous identity...", 180, ["umbra"]}
       - {:summarized_memories, req1, "previous discussion covered...", 120}
       - {:entities, evt_id, [%{name: "umbra", type: :person, sentiment: :engaged}]}

    7. Attentive layer (Haiku) receives pre-digested material:
       compressed thread + memory briefing + entity context.
       Total: ~400 tokens of dense context instead of ~2000 tokens raw.

    8. Haiku decides: confident → draft reply itself,
       OR low confidence → escalate to Engaged (Sonnet).

The Processor's entire purpose is step 7: turning 2000 tokens of raw
material into 400 tokens of signal. That's what makes API calls
efficient.

------------------------------------------------------------------------

## Self-Maintenance Loop

`groom_entities` and `analyze_rules` run on a cron schedule via
TaskScheduler (every \~6h):

-   Review recent interactions → suggest entity tier
    promotions/demotions
-   Review rules table → which rules worked, which were wasted, what
    patterns suggest new rules
-   Feed analysis to Sonnet next time it enters Engaged state

The system improves its own entity model and rule-writing quality not
through training but through operational reflection. The Processor does
the heavy analysis; Sonnet makes the judgment calls.

------------------------------------------------------------------------

## Recipe Type Signatures

Every recipe declares the capabilities it requires. The Processor is a
dispatcher: it matches a recipe's requirements against the capability
pool and either routes to a capable backend, queues for later, or
errors.

    %Recipe{
      name: :compress_thread,
      requires: [:summarization],
      ...
    }

    %Recipe{
      name: :extract_entities,
      requires: [:feature_extraction],
      ...
    }

    %Recipe{
      name: :draft_session_note,
      requires: [:reasoning],
      ...
    }

    %Recipe{
      name: :describe_image,
      requires: [:vision, :feature_extraction],
      ...
    }

If no backend currently satisfies a recipe's requirements:
- **Queue** the work if the capability exists in the pool but all
  providers are degraded (it'll recover)
- **Error** if no backend in the pool declares the capability at all
  (operator needs to add one)

The Processor never picks a specific backend. It says "I need
`:summarization`" and BackendPool hands it one. This is what makes
adding new capabilities painless — the Processor doesn't change when
you connect a vision model. You write a new recipe with
`requires: [:vision]` and it just works.

See `DESIGN-backend-capabilities.md` for the full capability graph.

------------------------------------------------------------------------

## Design Properties

-   **All operations are idempotent** --- safe to retry on failure
-   **All operations return token counts** --- caller knows the cost of
    using the result
-   **All operations are async** --- Processor manages its own queue,
    caller never blocks
-   **Recipes are typed** --- each declares required capabilities, not
    a specific backend. No capability = no dispatch, never a silent
    failure
-   **Processor never selects backends** --- it declares requirements,
    BackendPool resolves them. Adding a new modality is a recipe + a
    config file, not a Processor change
