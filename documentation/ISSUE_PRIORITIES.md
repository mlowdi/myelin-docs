# Issue Priorities

Assessed 2026-03-19. Review periodically as work progresses.

## P1 — Foundational, unblocks other work

### #12 — Implement Sessions
The biggest missing structural piece. Cost attribution, briefings, compaction boundaries, and audit all depend on sessions existing. The spec is already written (`SPEC-sessions.md`). Until sessions exist, the system can't properly track what the agent did during an escalation, which means no meaningful self-analysis, budgeting, or maintenance. Nearly everything else gets better once this lands.

### #3 — Async tool calling
The agent can't be genuinely useful until it can call tools and get results back. This is the difference between "chatbot that knows stuff" and "agent that does stuff." The two-tier model (sync for fast reads, async for slow operations) is the right design. Needs to be resolved before any serious interface work because every interface will need to handle async tool results.

### #9 — Permission objects
Safety-critical and architecturally foundational. The system is designed around bounded autonomy — but the bounds don't exist yet beyond `requires_approval?`. This needs to land before the agent gets any real capabilities, because retrofitting permissions onto an already-capable agent is how accidents happen. The structured permission request flow (agent asks, human approves, time-limited grant) is genuinely novel and worth getting right early.

## P2 — High-value, independent

### #35 — Batch summarization
The cost economics are compelling. Depends loosely on sessions (#12) for the "summarize on session close" flow, but the batch pipeline itself is independent. The draft/final two-tier quality model is smart. This is where the "cheap to run" promise becomes real.

### #8 — read_docs tool
Small, high leverage. Every model that can search internal docs makes fewer mistakes. Force multiplier for everything else — the agent understanding its own architecture means better routing, better tool use, better self-maintenance. Quick win.

### #4 — Prompt cache strategy (conversation timesharing)
Caching a warm layer of ongoing concerns and only injecting the ephemeral thread context uncached is excellent cost engineering. Compounds over time — every inference call gets cheaper. Should come after sessions since session lifecycle informs what gets cached.

## P3 — Valuable extensions

### #38 — Ignore/deprioritize entities
The "I know already, leave me alone" use case is real — especially for high-volume ingestion. But the decay inversion question and the permanent-ignore semantics need speccing out before implementation.

### #36 — Miniflux RSS interface
Good test of the Interface abstraction and a genuinely useful capability (passive discovery of interesting content). Independent of everything else. Can be parallelized with other work.

### #10 — Automated registry grooming
Maintenance infrastructure. Important eventually but the system works without it — things just get stale. Agents should be able to add but not remove — decay functions and automated processes handle cleanup. Depends on having enough running history to know what "stale" looks like.

## P4 — Research / future

### #37 — Entity normalization
The problem is real but huge. The "start small" approach is right — associate multiple identifiers with an entity and expand from there. Long-term research track, not a sprint item.

### #5 — Haiku narrative roles
Pure research question. The A/B test approach is right but needs sessions and cache strategy working first to run the experiment meaningfully.

## Milestone (not for immediate implementation)

### #39 — Unify inference clients behind Inference.* abstraction layer
Future architectural convergence. See issue for details and interim guardrails.

## Closed (2026-03-19)

- **#18** (Decouple InferenceRouter from LlamaClient) — subsumed by #39
- **#23** (Cache.Manager ETS state lost) — conscious tradeoff, not a bug; 5-min TTL means cache is ephemeral by design
- **#20** (Abstract ConversationRegistry persistence) — no concrete second backend; premature abstraction per ARCHITECTURE_CONSTRAINTS.md
- **#19** (Pluggable enrichment sources) — no concrete second enricher yet; enrichment happens at a choke point so easy to retrofit when needed (likely for SIEM/Zabbix use cases)
