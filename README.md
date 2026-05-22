# myelin-docs

Design specifications and reference documentation for **Myelin** — an Elixir/OTP agent runtime for AI agents with ongoing concerns. The architecture is designed to emphasize modularity with loose coupling between components, and economy by doing as much work as possible locally before sending calls to external APIs for inference.

The core architecture: mechanical filtering → cheap local enrichment and inference → expensive API inference (Haiku/Sonnet/Opus). Two-pass routing, pluggable backends, self-maintaining context.

---

## Repository layout

```
myelin-docs/
├── documentation/        # Reference docs — what the system is and how it works
└── design-spec-etc/      # Design history — specs, proposals, principles, rationale
```

---

## `documentation/` — Reference docs

Numbered sequence covering the full system, plus a few standalone files.

| File | Topic |
|------|-------|
| [00-system-map.md](documentation/00-system-map.md) | Architecture overview: core vs. pluggable components, data flow, dependency matrix, ETS/SQLite schema summary, type system overview |
| [01-event-system.md](documentation/01-event-system.md) | `%Event{}` struct, ingestion lifecycle (enrichment → scoring → queueing → routing), EventPipeline, salience scoring |
| [02-inference-routing.md](documentation/02-inference-routing.md) | Two-pass routing (vibe check + action decision), 5-state StateMachine, 4-tier inference capabilities, BackendPool, ContextBuilder, Personality, budget tracking |
| [03-persistence-memory.md](documentation/03-persistence-memory.md) | 3-layer memory (operational state / agent memory / knowledge providers), MemoryStore, KnowledgeStore with FTS5+Vec0, Cache.Manager, Engram integration |
| [04-interfaces-output.md](documentation/04-interfaces-output.md) | Interface Registry, Telegram/Bluesky/Syslog adapters, Interface behaviour contract, 8-stage OutputPipeline lifecycle, OutputEvent schema |
| [05-conversations-processing.md](documentation/05-conversations-processing.md) | `%Conversation{}` struct, ConversationRegistry with decay/tracking levels, compaction tiers, Processor FIFO queue with preemption, Recipes, TaskScheduler, Research threads |
| [06-design-rationale.md](documentation/06-design-rationale.md) | Cross-cutting principles: event philosophy, inference economics, async-first, self-maintaining context, temporal awareness, pluggability, observability |
| [07-secrets.md](documentation/07-secrets.md) | Centralized GenServer-based secrets, `.secrets`/`.env` file formats, API reference |
| [08-backend-config.md](documentation/08-backend-config.md) | TOML-based backend declaration in `config/backends/`, capability/cost/cache fields, startup loading |
| [09-inference.md](documentation/09-inference.md) | Unified inference abstraction: Request/Response/Receipt structs, dependency injection via `inference_mod`, budget recording |
| [09-type-reference.md](documentation/09-type-reference.md) | Lookup table for every core struct across events, conversations, interfaces, and infrastructure |
| [10-interaction-patterns.md](documentation/10-interaction-patterns.md) | Module-pair communication patterns, PubSub topics, Task patterns, error propagation |
| [10-interactive-memory-tools.md](documentation/10-interactive-memory-tools.md) | Ephemeral session-scoped state (todos/scratchpad), Layer 3 context injection, tool execution loop for Interactive sessions |
| [11-admin-command-channel.md](documentation/11-admin-command-channel.md) | Principal/role model, CommandChannel, MVP command set (`/status`, `/backends`, `/budget`, `/kick`), registration flow |
| [12-miniflux.md](documentation/12-miniflux.md) | Bidirectional RSS aggregator integration: polling flow, starring flow, configuration |
| [ISSUE_PRIORITIES.md](documentation/ISSUE_PRIORITIES.md) | P1–P4 classification of 39 open issues (assessed 2026-03-19) |
| [README.md](documentation/README.md) | Index with navigation links to all 11 subsystems |

---

## `design-spec-etc/` — Design history

Source of truth for *why* the system is designed the way it is. Organized by document type.

### Branding

| File | Description |
|------|-------------|
| [NAME.md](design-spec-etc/NAME.md) | Name rationale (myelin sheath → signal insulation), namespace conventions, tagline |

### SPEC — Detailed subsystem specifications

| File | Description |
|------|-------------|
| [SPEC-event-schema.md](design-spec-etc/SPEC-event-schema.md) | `%Event{}` schema design: identity/classification/content/signals/metadata fields, ingestion cost model, design decisions |
| [SPEC-state-machine.md](design-spec-etc/SPEC-state-machine.md) | 5-state machine (Dormant → Monitoring → Attentive → Engaged → Interactive), threshold policies, guard conditions |
| [SPEC-router-prompts.md](design-spec-etc/SPEC-router-prompts.md) | Two-pass routing prompts: Pass 1 vibe check (~3s), Pass 2 routing decision (~5s), grammar-constrained output (GBNF), template management |
| [SPEC-rules-table.md](design-spec-etc/SPEC-rules-table.md) | Dynamic routing reprogramming via Sonnet-written rules: schema, rule kinds (boost/suppress/pin/escalate/custom), guardrails |
| [SPEC-sessions.md](design-spec-etc/SPEC-sessions.md) | Session as unit of "the system did a thing": lifecycle (opened → closed → compacted), cost tracking per session, audit/briefing use cases |
| [SPEC-conversation.md](design-spec-etc/SPEC-conversation.md) | `%Conversation{}` as first-class abstraction over thread_id: ConversationRegistry, lifecycle, tracking levels with decay, compaction policy, novelty scoring |
| [SPEC-cache-strategy.md](design-spec-etc/SPEC-cache-strategy.md) | Cache strategy v1: context zone architecture (frozen/hot/ephemeral), token budgets per tier, cache window lifecycle |
| [SPEC-cache-strategy-v2.md](design-spec-etc/SPEC-cache-strategy-v2.md) | Cache strategy v2 (supersedes v1): 4 breakpoints with mixed TTLs, layered frozen zone, divergence scoring, session ↔ cache layer relationship |
| [SPEC-interfaces.md](design-spec-etc/SPEC-interfaces.md) | Interface behaviour (capabilities/deliver/can_deliver?), OutputEvent schema, capability manifests, 8-stage output pipeline, Interface Registry |
| [SPEC-processor.md](design-spec-etc/SPEC-processor.md) | Processor atom vocabulary: memory ops, routing support, content prep, maintenance — concrete flow example, self-maintenance loop |
| [SPEC-research-threads.md](design-spec-etc/SPEC-research-threads.md) | Haiku as gate+research agent: thread lifecycle, triggers, tool access, budget controls, engram storage |
| [SPEC-interactive-prompt.md](design-spec-etc/SPEC-interactive-prompt.md) | Interactive session prompt system: structured output schema, session narrative caching/rewrite, hot zone construction |
| [SPEC-tool-system.md](design-spec-etc/SPEC-tool-system.md) | Sync tools (API-declared, blocking, inform this response) vs. async tools (structured output, fire-and-forget, change the world) |
| [SPEC-speculative-computation.md](design-spec-etc/SPEC-speculative-computation.md) | Pi and VM as sunk costs — maximize Processor saturation during quiet hours: task types, scheduling, payoff model |

### DESIGN — Proposals and migration plans

| File | Description |
|------|-------------|
| [DESIGN-inference-unification.md](design-spec-etc/DESIGN-inference-unification.md) | Unified inference abstraction decoupling callers from specific backends: Request/Receipt/Response API, locality constraints, 4-phase migration |
| [DESIGN-inference-tiers.md](design-spec-etc/DESIGN-inference-tiers.md) | 4-tier capability model (`:edge`/`:efficient`/`:capable`/`:frontier`), routing strategies (prefer_cost/speed/quality), caller audit, Wave A–D migration |
| [DESIGN-backend-capabilities.md](design-spec-etc/DESIGN-backend-capabilities.md) | Routing as performance contract, primitive/composite capabilities, startup dependency resolution, deployment profiles (Openclaw, headless, budget, full-local, kitchen sink) |
| [DESIGN-deterministic-routing.md](design-spec-etc/DESIGN-deterministic-routing.md) | Hybrid Pass 1 (fast-path + LLM fallback) and fully deterministic Pass 2 (pure Elixir), TOML-based routing config, Wave 1–2 migration |
| [DESIGN-pluggable-memory.md](design-spec-etc/DESIGN-pluggable-memory.md) | Three-layer memory model: `Myelin.Memory` and `Myelin.Knowledge` behaviours, interaction flows, Wave 1–4 migration (DONE) |
| [DESIGN-config-standardization.md](design-spec-etc/DESIGN-config-standardization.md) | Unified TOML loader across backends/interfaces/knowledge, secret handling, validation patterns, migration (DONE) |
| [DESIGN-dependency-resolution.md](design-spec-etc/DESIGN-dependency-resolution.md) | Declarative Provider Manifest (Provides/Requires/Constraints), operable/degraded/inoperable states — future work, not yet built |
| [DESIGN-temporal-context.md](design-spec-etc/DESIGN-temporal-context.md) | Principle for surfacing temporal relations in prompts: weekday + ISO + readable delta, token economy |

### IMPL — Implementation plans

| File | Description |
|------|-------------|
| [IMPL-conversations.md](design-spec-etc/IMPL-conversations.md) | Phased build plan for Conversations (C1–C4), key decisions (`conversation_id == thread_id`), phase status |

### PRINCIPLE & VISION

| File | Description |
|------|-------------|
| [PRINCIPLE-context-as-scratchpad.md](design-spec-etc/PRINCIPLE-context-as-scratchpad.md) | Context is a manufactured briefing (~80% deterministic, ~20% inference), not a literal transcript. Context budget as design constraint. |
| [VISION-beyond-social.md](design-spec-etc/VISION-beyond-social.md) | Myelin as general-purpose attention architecture: SOC/SRE/support/research/ops — the interfaces change, the stack doesn't |

### GUIDE

| File | Description |
|------|-------------|
| [GUIDE-manpages.md](design-spec-etc/GUIDE-manpages.md) | How to add `.md` files to `priv/manpages/` for `Myelin.ReadDocs`: directory structure, format rules, no registration required |

---

## Topics at a glance

| Topic | Reference doc | Design/spec |
|-------|---------------|-------------|
| Event ingestion & salience | `01-event-system.md` | `SPEC-event-schema.md` |
| Routing & state machine | `02-inference-routing.md` | `SPEC-state-machine.md`, `SPEC-router-prompts.md`, `DESIGN-deterministic-routing.md` |
| Inference backends | `09-inference.md`, `08-backend-config.md` | `DESIGN-inference-unification.md`, `DESIGN-inference-tiers.md`, `DESIGN-backend-capabilities.md` |
| Prompt caching | `02-inference-routing.md` | `SPEC-cache-strategy-v2.md`, `SPEC-cache-strategy.md` |
| Memory & knowledge | `03-persistence-memory.md` | `DESIGN-pluggable-memory.md` |
| Conversations | `05-conversations-processing.md` | `SPEC-conversation.md`, `IMPL-conversations.md` |
| Sessions | — | `SPEC-sessions.md` |
| Output & interfaces | `04-interfaces-output.md` | `SPEC-interfaces.md` |
| Interactive mode | `10-interactive-memory-tools.md` | `SPEC-interactive-prompt.md`, `SPEC-tool-system.md` |
| Async processing | `05-conversations-processing.md` | `SPEC-processor.md`, `SPEC-research-threads.md`, `SPEC-speculative-computation.md` |
| Dynamic rules | — | `SPEC-rules-table.md` |
| Admin & ops | `11-admin-command-channel.md`, `07-secrets.md` | — |
| Configuration | `08-backend-config.md` | `DESIGN-config-standardization.md`, `DESIGN-dependency-resolution.md` |
| Design principles | `06-design-rationale.md` | `PRINCIPLE-context-as-scratchpad.md`, `DESIGN-temporal-context.md`, `VISION-beyond-social.md` |
