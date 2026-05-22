# Beyond Social — Myelin as General-Purpose Attention Architecture

*What started as "give Liv a presence on Bluesky" turns out to be a general-purpose framework for intelligent event triage, context assembly, and human augmentation.*

---

## The Realization

Myelin's core architecture — mechanical filtering → cheap local inference → expensive API inference → human judgment — is not specific to social media. It's a general solution to a general problem: **too many signals, not enough attention.**

Every domain that deals with event streams has this problem:
- SOC analysts drowning in alerts
- SREs reacting to infrastructure events
- Customer support teams triaging tickets
- Researchers tracking literature across dozens of feeds
- Operations teams managing heterogeneous environments

The architecture doesn't change. The interfaces do.

---

## The Stack

```
┌─────────────────────────────────────────────────────┐
│  HUMAN                                              │
│  Institutional knowledge, gut feeling, judgment     │
│  calls that require full context no model has.      │
│  Receives: pre-processed information, not raw data  │
├─────────────────────────────────────────────────────┤
│  JUDGMENT LAYER  (API models — Sonnet/Opus)         │
│  "Should we escalate? Is this related to last       │
│  week's change? Draft a response."                  │
│  Cost: expensive. Reserved for when it matters.     │
├─────────────────────────────────────────────────────┤
│  PROCESSOR LAYER  (local 4B — Processor)            │
│  Summarize, classify, extract state, draft.         │
│  Cost: cheap, seconds, runs on local hardware.      │
├─────────────────────────────────────────────────────┤
│  MECHANICAL LAYER  (deterministic — no inference)   │
│  Counting, correlating, keyword matching,           │
│  threshold checks, salience scoring, conversation   │
│  tracking, entity lookup, trend detection.          │
│  Cost: free, instant, deterministic.                │
└─────────────────────────────────────────────────────┘
```

Each layer does what it's best at. Nothing reaches the layer above without being processed by every layer below. **The human gets information, not data.**

---

## Why This Works

### 1. Most events don't matter

In any event stream — security alerts, social media, infrastructure monitoring — the vast majority of events are noise. Myelin's architecture is built around this truth: the mechanical layer drops 90%+ of events before any inference is spent. The router (2B local model) handles another chunk. By the time an API model sees something, it's already been pre-qualified as worth the tokens.

### 2. Context compounds

Conversations (threads, incident timelines, research sessions) accumulate salience over time. Individual events that wouldn't cross threshold alone become significant as part of a pattern. The conversation tracking system does this mechanically — counting, trending, correlating — before any model gets involved.

### 3. Memory shapes attention

Novelty-weighted compaction means the system develops genuine "interests" over time. Topics the user cares about build dense engram clusters. New information in those areas gets noticed faster (keyword hits, hot topic matches). Information that's redundant with existing knowledge gets deprioritized. The agent naturally becomes more attentive to what matters and less distracted by what doesn't.

### 4. The expensive stuff is reserved for judgment, not comprehension

Most "AI agent" systems shove raw context into a large model and ask it to figure everything out. Myelin inverts this: by the time a large model sees anything, comprehension is already done. The model's job is *judgment* — "should we act on this?" "is this related to that?" "what should we say?" — which is the correct use of expensive tokens.

---

## Application Domains

### Social presence (the original use case)
- **Interfaces:** Bluesky, Telegram, IRC/Matrix
- **Conversations:** Threads, DMs, group chats
- **Value:** Intelligent timeline parsing, relationship tracking, contextual engagement
- **Human role:** Creative direction, relationship decisions, voice

### SRE / Infrastructure operations
- **Interfaces:** Zabbix/Prometheus (alerts), SSH/bash (diagnostics), PagerDuty (escalation)
- **Conversations:** Incident timelines, diagnostic sessions, deployment correlations
- **Value:** Automated context assembly — by the time the SRE picks up their phone, the agent has correlated the alert with recent deployments, pulled relevant logs, checked runbooks, and drafted a summary
- **Human role:** Root cause judgment ("this is actually the new customer's traffic pattern hitting a latency timeout"), rollback decisions, architectural fixes

### SOC / Security operations
- **Interfaces:** SIEM (alerts), syslog (events), threat intel feeds, ticketing
- **Conversations:** Alert clusters, investigation threads, incident timelines
- **Value:** Mechanical triage (the grunt work), log correlation, threat intel enrichment, IOC extraction — the analyst gets a briefing, not a queue
- **Human role:** Attribution, impact assessment, response strategy, communication

### MSP / Service delivery
- **Interfaces:** Ticketing system, email, M365 service health, customer portals
- **Conversations:** Ticket threads, change request chains, customer communication
- **Value:** Cross-customer pattern detection ("three customers reported the same Teams issue this morning"), SLA tracking, documentation lookup, change correlation
- **Human role:** Customer relationship, prioritization across competing demands, escalation judgment

### Research / Knowledge work
- **Interfaces:** RSS feeds, arXiv, social media, tool chains (web search, document fetch)
- **Conversations:** Literature threads, research sessions, citation chains
- **Value:** Novelty detection against existing knowledge base, automated literature summarization, "this new paper contradicts something you cited last month"
- **Human role:** Evaluation, synthesis, original thinking

---

## The Key Insight

**Myelin doesn't replace the human in the loop. It gives them a head start.**

A good SOC analyst doesn't need an AI to tell them what to do. They need an AI that's spent the last five minutes assembling context so they can make the right call in thirty seconds instead of five minutes. A good SRE doesn't need an AI to roll back a deployment. They need an AI that's already correlated the alert with the deployment, checked the logs, and confirmed the rollback runbook exists — so they can decide in seconds whether rollback is the right call.

The architecture works because it mirrors how expert humans already think: filter mechanically, build context progressively, reserve expensive cognition for judgment calls. Myelin just does the first three steps faster and never forgets to check the runbook.

---

## What Doesn't Change Across Domains

- Event schema (`%Event{}`) — universal, platform-normalized
- Conversation tracking — threads/timelines/sessions are the same type everywhere
- Salience scoring — mechanical + conversation + entity + topic signals
- Compaction policy — salience gets you noticed, novelty gets you remembered
- Cache hierarchy — identity (cold) → situational (warm) → session (hot)
- Cost-aware inference routing — mechanical → local 4B → local 2B router → API

## What Changes Per Domain

- Interfaces (the platform adapters)
- Entity registries (who/what matters in this domain)
- Hot topics (what the user/organization currently cares about)
- Rules table (domain-specific routing overrides)
- Output channels (where results are delivered)
- Conversation type vocabulary (domain-specific classification)

The core is the same. The periphery adapts. That's the whole point of the interface abstraction — stamp at the edge, reason in the center.

---

## Status Dashboard

The system's internal state is inherently observable — ETS tables, conversation registries, salience scores, session states, rule tables, cache windows. A real-time dashboard that exposes this makes the architecture tangible.

### What it shows

**Conversation stream** — conversations appearing, rising through tracking levels, triggering sessions. Watch a thread go from `:untracked` → `:watched` → `:active` as salience accumulates. See the moment the router escalates and a Sonnet session opens.

**Salience heatmap** — live salience scores across active conversations. Rising trends pulse. Conversations that cross threshold flash. The mechanical → processor → judgment pipeline visualized as events flow through layers.

**Session timeline** — when sessions open, what triggered them, what inferences were run, what outputs were generated, total cost. A single session expanding into its activity: "opened because thread:xyz crossed salience 0.7, ran 2 Processor calls and 1 Sonnet call, produced 1 reply and 1 keyword rule, cost $0.04, closed after 8 minutes."

**Register state** — the hot topics register, entity registry, active rules table, cache windows. Live updates as rules are written, entities are promoted/demoted, cache windows open and close.

**Cost tracker** — per-session, per-conversation, per-hour, per-day. Which conversations are expensive? Which are cheap? Where is the budget going?

### Why it matters

This isn't vanity metrics. This is the "SOC dashboard" for the agent — the operator (Martin, initially) can see exactly what the system is paying attention to, why, and at what cost. When something goes wrong (false positive escalation, missed event, budget spike), the dashboard shows the causal chain.

It's also the demo. When people see conversations bubbling through salience layers in real time, research tasks spawning from a Sonnet session, keyword rules appearing and immediately affecting subsequent event scoring — that's when the architecture clicks. The spec files explain the *what*. The dashboard shows the *how*.

### Implementation sketch

Phoenix LiveView is the obvious choice — Elixir native, real-time via WebSocket, OTP supervision integration. The data is already in ETS and SQLite. A LiveView process subscribes to PubSub topics (`:event_ingested`, `:session_opened`, `:conversation_updated`, etc.) and renders updates in real time.

This is a later-phase addition — the system works without it. But it's the thing that makes the system *legible*.

---

## Scaling: More Local Compute, Not More API Tokens

Every other agent framework's scaling story is "buy more API tokens." Myelin's is: **increase local compute and make API calls rarer and more valuable.**

The architecture is layered by cost. Every time you strengthen a lower layer, the layers above get called less often.

### The progression

```
Phase 1 — Pi 5 (today):
  Router:     2B (llama-server on Pi 5)     → catches ~90% of noise
  Processor:  4B (llama-server on VM)       → summarize, classify, simple tasks
  Judgment:   API (Haiku/Sonnet/Opus)       → everything that needs real intelligence

Phase 2 — dedicated GPU (rented or owned):
  Router:     8B                            → catches ~95%, handles nuanced routing
  Processor:  16-24B                        → complex summarization, conversation
                                              evaluation, tool session oversight
  Workers:    task-specific finetunes       → SSH diagnostics, log parsing, code review
  Judgment:   API                           → genuinely hard judgment calls only

Phase 3 — multi-GPU or cloud burst:
  Router:     8B (dedicated, always warm)
  Processor:  24-32B (primary GPU)
  Workers:    multiple finetunes time-sharing secondary GPU
  Judgment:   API                           → rare, expensive, high-value only
```

At each phase, the API bill *drops* as local capability *rises*. The architecture doesn't change — the same event pipeline, the same conversation tracking, the same salience scoring. The models just get more capable at each tier.

### GPU saturation: idle compute is wasted compute

If the Processor is a 24B model on a rented GPU, it's idle between sessions. That idle time is paid for. Fill it with background work:

- **Speculative compaction** — process old conversations that haven't been compacted yet
- **Deep topic extraction** — richer keyword/embedding analysis on `:watched` conversations
- **Reclassification sweeps** — re-evaluate conversation types as more data accumulates
- **Engram maintenance** — consolidate redundant memories, identify stale knowledge, merge overlapping topic clusters
- **Pre-computation** — embeddings for entity registry updates, summary refreshes for active conversations
- **Speculative research** — "we have 40 minutes of idle GPU before the next likely escalation, let's deep-read those three papers the user bookmarked"

All of this is useful work that makes the system smarter over time. The TaskScheduler already understands cache windows and opportunistic scheduling — this is the same concept applied to compute capacity instead of prompt cache TTLs.

### Finetunes as specialists

A model fine-tuned on "parse Linux diagnostic output and extract structured findings" doesn't need to be large. It needs to be good at one thing. The conversation architecture makes this natural:

1. The Processor classifies the conversation type (`:maintenance`, `:research_session`, etc.)
2. The conversation type maps to a preferred worker model
3. The worker executes in its own conversation, reports back
4. The judgment tier evaluates the result

Multiple specialized 7-8B finetunes can time-share a single GPU. The conversation's type tag — already computed for compaction policy and dashboard display — becomes the routing signal. One classification, three uses.

```
GPU time-sharing on a single rented A100:

  ┌─────────────────────────────────────────────────┐
  │ 24B Processor (primary, always loaded)          │
  │ handles: summarization, classification,         │
  │ conversation evaluation, context assembly       │
  ├─────────────────────────────────────────────────┤
  │ 7B sysadmin finetune (loaded on demand)         │
  │ handles: terminal sessions, log parsing,        │
  │ system diagnostics, SMART report interpretation  │
  ├─────────────────────────────────────────────────┤
  │ 7B research finetune (loaded on demand)         │
  │ handles: web content extraction, document       │
  │ parsing, citation tracking, literature review   │
  ├─────────────────────────────────────────────────┤
  │ 7B code finetune (loaded on demand)             │
  │ handles: code review, refactoring suggestions,  │
  │ architecture analysis, test generation          │
  └─────────────────────────────────────────────────┘

  Load latency: ~2-5 seconds for a 7B model swap.
  Acceptable for async task conversations.
  The Processor stays resident; specialists swap in/out.
```

### The economic inversion

Traditional agent cost structure: **capability scales with API spend.** More complex tasks = more tokens = higher bill = linear cost growth.

Myelin cost structure: **capability scales with local compute.** More complex tasks = more local processing = fixed infrastructure cost. API calls reserved for judgment. As local models improve (and they improve every quarter), the same hardware handles more complex work. The API bill trends *down* over time even as capability trends *up*.

This is the same economic model as running your own SOC vs. outsourcing to an MSSP. The upfront investment is higher, but the marginal cost of each additional "analyst action" is near zero once the infrastructure is in place.

---

*Started as a bluesky bot. Turns out it's twelve years of SOC experience crystallized into an architecture.*
