# Design Rationale

This document captures the core design principles of the Myelin architecture and the specific rationale behind key architectural decisions. The system is designed to handle high-volume event streams by progressively filtering noise and reserving expensive inference only for situations requiring genuine judgment.

## 1. Event Philosophy: Stamp at Edge, Reason in Center

**Principle:** The system normalizes heterogeneous inbound signals into a single, unified `%Event{}` schema at the system boundary. All downstream reasoning relies exclusively on this standardized data carrier.

**Why:** Different platforms (Bluesky, Telegram, Syslog, Webhooks) have wildly varying structures. If the core reasoning and routing engines had to understand platform-specific details, the system would be brittle and difficult to extend. By doing the cheap, deterministic work (entity lookup, embedding generation, keyword matching) at the edge and stamping it onto the event, the core remains strictly focused on triage and escalation.

**Concrete Example:** The Bluesky interface receives an ATProto post. It doesn't send the raw JSON to the router. Instead, it performs a local database lookup for the user, runs a semantic match against hot topics, and emits an `%Event{}`. The local 2B router model only sees abstract signals like `sender_tier: 2`, `topic_match: 0.7`, and `kind: reply`. It never sees the raw API payload.

## 2. Conversation Architecture: Conversations as Composition

**Principle:** A conversation is a first-class abstraction for *any* stateful, multi-turn exchange. Social media threads, bash terminal sessions, and multi-step tool chains are all treated as the exact same data type.

**Why:** Grouping events by a thread identifier allows salience and context to compound over time. More importantly, treating internal tool chains as conversations eliminates the need for an embedded orchestration DSL. The "program execution trace" *is* the conversation record, making complex behaviors like branching, nested delegation, and error handling emergent properties of the conversation tracking system. It also establishes strict scope isolation between different intelligence tiers.

**Concrete Example:** When Sonnet (the judgment tier) delegates a diagnostic task to a cheaper 32B worker model, the worker runs an SSH session in its own isolated `:bash` conversation. The worker executes commands and reads noisy `iostat` outputs within its own scope. It then synthesizes the findings and posts a summary back into the parent `:tool_chain` conversation. Sonnet only ever reads the clean summary, isolating the expensive judgment model from the raw, high-token data.

## 3. Inference Economics: Hierarchy, Caching, and Budgets

**Principle:** Expensive, powerful API inference is treated as a rare, precious resource. The system utilizes deterministic code, free local models, and layered context caching to minimize API spend.

**Why:** Naive LLM agents spend thousands of tokens constantly re-reading transcripts and processing noise, resulting in API costs that scale linearly with inbound activity. By mirroring CPU cache architectures (L1 = Hot zone, L2 = Prompt cache, L3 = Embeddings) and routing requests based on task complexity, the system remains economically viable at scale.

**Concrete Example:** The system employs a "Two-Pass Local Routing" strategy. A local 2B model runs a cheap (~3 second) Pass 1 "vibe check" returning only `quiet`, `notable`, or `urgent`. If an event is notable, it proceeds to Pass 2 for an action decision. Only if Pass 2 decides to `escalate` does the system assemble context and call an API model. To further reduce cost, stable information like the agent's identity and behavioral constraints are packaged into a Layer 0 block with a 1-hour cache TTL. This cache is kept alive automatically by subsequent calls, reducing the daily frozen-context cost by over 90%.

## 4. Async-First: Optimistic Execution

**Principle:** Operations that change the environment or have unbounded latency (like web searches) are executed asynchronously and optimistically via structured output. Native, blocking API `tool_use` is reserved exclusively for fast, read-only memory operations.

**Why:** Blocking generation while waiting for a 15-second web fetch ruins system throughput and interactivity. The system needs to be able to fire off tasks and move on, trusting that results will be provided when ready. 

**Concrete Example:** When the model decides to run a web search, it does not use a synchronous API tool. Instead, it issues an async command in its structured output: `{"op": "web_search", "failure_mode": "next_turn"}`. The system dispatches the task and the current turn completes. The model assumes success and moves on. When the search completes, the system injects the results into the hot zone of the *next* natural turn. If the search fails, the system handles the notification based on the declared `failure_mode`.

## 5. Self-Maintaining Context: The Scratchpad Narrative

**Principle:** The active context window is a manufactured briefing, not an ever-growing literal transcript. The model uses structured output to continuously compress and maintain its own working memory.

**Why:** Literal transcripts grow linearly, diluting the signal-to-noise ratio and wasting tokens on conversational pleasantries. By forcing the model to summarize its own actions as it goes, the context window remains dense and bounded, acting as a functional scratchpad rather than a historical log.

**Concrete Example:** During an Interactive session, the model's output schema requires both a `response_text` (what the user actually sees) and a `narration` field. The model uses the `narration` field to write a 1-3 sentence, first-person summary of the exchange (e.g., "User asked about feature X; I explained Y"). This dense narration replaces the raw transcript in the session's hot zone. Over 20 turns, instead of carrying 4,000 tokens of raw chat logs, the model context only carries about 700 tokens of highly relevant session memory.

## 6. Temporal Awareness: Deltas over Timestamps

**Principle:** Temporal relationships are explicitly calculated and presented as relative deltas rather than raw ISO timestamps.

**Why:** LLMs are language models, not math models. They cannot reliably subtract two raw timestamps to deduce pacing, urgency, or delays. The system must translate chronological data into semantic meaning so the models can "feel" the rhythm of a conversation.

**Concrete Example:** Instead of feeding the router a raw timestamp like `2026-03-16T14:30:00Z`, the prompt builder pre-computes the delta and injects `last message in thread: 2m ago`. For richer context assembly, it provides both: `Monday 2026-03-16T14:30:00+01:00 (1 day, 15h 12m since last message)`. 

## 7. Pluggability: Interfaces and Backend Abstraction

**Principle:** The external boundaries of the system—both where events originate and where compute happens—are entirely decoupled from the core routing and reasoning logic. 

**Why:** The core value of the architecture is its cognitive triage pipeline. Tying the core directly to specific platforms or specific AI providers prevents the system from adapting to new hardware or new domains (e.g., pivoting from social media to SIEM security alerts). 

**Concrete Example:** To add a new platform like Telegram, a developer creates an `Myelin.Interface.Telegram` module that defines a `capabilities()` manifest. This manifest declares constraints (e.g., `max_length: 4096`, `formats: [:markdown]`). The system renders this manifest directly into the 1-hour cached situational context layer. The model reads this text and instantly knows it can output to Telegram using the declared constraints, without requiring any new hardcoded logic in the core reasoning engine. Similarly, adding a local 70B GPU model simply requires registering it in the `BackendPool`; the router will automatically start assigning it tasks based on its latency and capability profile.

## 8. Observability Through Narrative

**Principle:** System logs, audit trails, and agent activity records are written as human-readable narrative, not structured log lines. The agent describes what it did and why in plain language, not in Common Log Format.

**Why:** A log you want to read is a log that gets read. Structured log formats are optimized for machines — they are compact, unambiguous, and queryable. But when a human needs to understand what an agent *did* and *why*, structured logs require translation. Narrative logs are immediately comprehensible: they convey subject, action, intent, and outcome in a form humans process naturally. An audit trail that reads like a story can be reviewed by an operator, a customer, or a regulator without tooling. A log full of `WARN: salience_threshold_exceeded entity=0x42 score=0.73` cannot.

**Concrete Example:** Instead of `INFO: routed event e_0x8f3a to backend anthropic-sonnet, action=react, salience=0.81`, the activity log records: "A reply from umbra arrived in an active thread about discontinuous identity. Salience was high and rising — I decided to engage and drafted a response." Instead of `DEBUG: processor recipe summarize_conversation triggered for conv_0x2b`, it records: "The thread had gone quiet for two hours. I summarized it while idle so the context would be ready if it picked back up." The event happened, the decision was made, the reason is preserved — all in one sentence a human can read.