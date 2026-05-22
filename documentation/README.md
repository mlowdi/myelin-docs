# Myelin Documentation Index

Welcome to the Myelin documentation. This directory contains detailed specifications of the agent's architecture, subsystems, and data flows.

### [00. System Map](./00-system-map.md)
High-level architectural overview, core vs. pluggable analysis, and dependency matrix.

### [01. Event System](./01-event-system.md)
Ingestion, enrichment, salience scoring, and the lifecycle of signals within the pipeline.

### [02. Inference & Routing](./02-inference-routing.md)
Two-pass routing (vibe check), attention states, state machine transitions, inference tiers, and backend pool management.

### [03. Persistence & Memory](./03-persistence-memory.md)
The 3-layer memory architecture, SQLite schemas, ETS tables, KnowledgeStore (FTS5/Vec), and external semantic memory integration.

### [04. Interfaces & Output System](./04-interfaces-output.md)
Input/output adapters (Telegram, Bluesky, Syslog), capabilities registry, and the delivery pipeline.

### [05. Conversations, Processing & Scheduling](./05-conversations-processing.md)
Conversation lifecycle, tracking levels, asynchronous processor recipes, and task scheduling.

### [06. Design Rationale](./06-design-rationale.md)
Cross-cutting design principles synthesized from specs: event philosophy, conversations as composition, inference economics, async-first, self-maintaining context, temporal awareness, pluggability.

### [07. Secrets Management](./07-secrets.md)
Secure handling of API keys, platform tokens, and credentials via process-isolated state.

### [08. Backend Configuration](./08-backend-config.md)
TOML-based backend declaration, capability mapping, cost tracking, and performance metrics.

### [09. Inference Unification](./09-inference.md)
Unified capability-based abstraction, canonical Request/Response/Receipt structs, and locality-based routing.

### [10. Type Quick-Reference](./09-type-reference.md)
All key structs with fields, types, descriptions, and data flow paths.

### [11. Interaction Patterns](./10-interaction-patterns.md)
Module communication mechanisms: call/cast/PubSub/Task patterns, topic registry, error propagation.
