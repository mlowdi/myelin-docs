# Myelin System Map

This document provides a high-level architectural overview of Myelin, classifying its components and mapping the flow of data through the system.

## 1. High-Level Architecture

```text
      [ External Platforms ]
      (Telegram, Bluesky, etc.)
               |
               v
      +------------------------+
      |      Interfaces        | <--- [ Interface.Registry ]
      +-----------|------------+
                  | %Event{}
                  v
      +------------------------+      +------------------------+
      |     EventPipeline      | ---> |      MemoryStore       |
      | (Filter, Enrich, Score)| <--- | (SQLite: myelin.db)    |
      +-----------|------------+      +------------------------+
                  | Enriched Event            ^          |
                  v                           |          |
      +------------------------+      +-------|----------|-----+
      |    InferenceRouter     | <--- |  ConversationRegistry  |
      | (Vibe Check, Action)   |      |  (ETS + Persistence)   |
      +-----------|------------+      +------------------------+
                  | Decision                  ^
                  v                           |
      +------------------------+      +-------|----------------+
      |      StateMachine      |      |       Processor        |
      | (Dormant -> Engaged)   | <--- | (Llama 4B: async work) |
      +-----------|------------+      +------------------------+
                  | %OutputEvent{}
                  v
      +------------------------+      +------------------------+
      |     OutputPipeline     |      |   Memory & Knowledge   |
      | (Validate, Rate Limit) |      | (Pluggable Behaviours) |
      +-----------|------------+      +------|-------|---------+
                  |                    Myelin.Memory  Myelin.Knowledge
                  v                   Memory.Grooming
      [ External Platforms ]
```

## 2. CORE vs PLUGGABLE Analysis

### Core Components (The "Engine")
Essential to Myelin's autonomous lifecycle and state management.
- **`Inference Layer`**: Unified capability-based abstraction for all LLM interactions. Decouples callers from specific backend wire formats and handles automated budget recording. Supports a 4-tier routing system (edge, efficient, capable, frontier).
- **`EventPipeline`**: Ensures all signals are normalized, filtered, and scored before reaching the router.
- **`InferenceRouter`**: The "brain's" gatekeeper. Implements the critical two-pass routing logic.
- **`StateMachine`**: Manages the agent's internal attention economy and behavioral posture.
- **`MemoryStore`**: The ultimate source of truth for persistent state and rules.
- **`ConversationRegistry`**: Owns the hot-path state and tracking levels of all active exchanges.
- **`Elixir/OTP Runtime`**: Provides the supervision, concurrency, and PubSub infrastructure.

### Pluggable Components (The "Limbs & Sensors")
Components designed to be swapped or extended without breaking the core engine.
- **`Interfaces`**: Platform adapters (Telegram, Bluesky, Syslog). They must implement the `Myelin.Interface` behaviour.
- **`Inference Backends`**: Model endpoints (local or remote) resolved via `BackendPool`.
- **`Memory Providers`**: `EngramClient` (semantic memory) and `KnowledgeStore` (independent document store) act as external knowledge bases.

### Boundary & Coupling Concerns
- **Inference Decoupling**: The introduction of the unified `Inference` layer removes direct coupling between system components (Router, Processor, Engaged) and specific leaf clients (`LlamaClient`, `AnthropicClient`).
- **Enrichment Coupling**: `EventEnricher` is hardcoded to consult `MemoryStore` and `ConversationRegistry`. Adding a new enrichment source (e.g., a real-time web search) requires modifying the core pipeline.
- **Persistence Chain**: `ConversationRegistry` flushes operational state to `MemoryStore`. Agent memory and knowledge access are decoupled via `Myelin.Memory` and `Myelin.Knowledge` behaviours, replacing hardcoded providers.

## 3. Data Flow Map

1.  **Ingestion**: An **Interface** (e.g., `:bluesky`) receives a message and constructs a `%Myelin.Event{}`.
2.  **Filtering**: `InputFilter` scans the payload for safety/canary strings.
3.  **Enrichment**: `EventEnricher` performs parallel lookups (Entity Tier, Keywords, Hot Topics, Conversation Context).
4.  **Initial Scoring**: `Salience.score/1` assigns a base value based on enrichment signals.
5.  **Queueing**: `EventPipeline` manages back-pressure, dropping low-salience events if the queue is full.
6.  **Routing**: `InferenceRouter` applies multiplicative rules and performs two-pass LLM routing:
    - **Pass 1 (Vibe Check)**: Is it `:notable`?
    - **Pass 2 (Action Decision)**: Should we `:react`, `:watch`, or `:ignore`?
7.  **State Evaluation**: `StateMachine` receives the decision. If salience crosses a threshold, it may transition state (e.g., `Attentive` -> `Engaged`).
8.  **Action Initiation**: In an active state, a response is generated (via `AnthropicClient`). An `%OutputEvent{}` is created.
9.  **Delivery Pipeline**: `OutputPipeline` validates, checks constraints, enforces rate limits, and waits for human approval if required.
10. **Delivery**: The target **Interface** executes the platform-specific API call (`deliver/1`).
11. **Feedback**: The delivery receipt is recorded in `ConversationRegistry` to maintain the agent's short-term memory of its own actions.

## 4. Dependency Matrix

| Module | Requires |
| :--- | :--- |
| **`Inference`** | `BackendPool`, `Budget`, `AnthropicClient`, `LlamaClient` |
| **`EventPipeline`** | `InputFilter`, `EventEnricher`, `Salience`, `PubSub` |
| **`InferenceRouter`** | `Salience`, `MemoryStore`, `Inference`, `BackendPool` |
| **`StateMachine`** | `InferenceRouter`, `PubSub`, `Attentive`, `Engaged` |
| **`ConversationRegistry`** | `MemoryStore`, `ETS`, `Conversation.Lifecycle`, `Conversation.Archiver` |
| **`MemoryStore`** | `Exqlite` |
| **`Memory.Grooming`** | `Memory`, `ThresholdDetector`, `Evaluator` |
| **`OutputPipeline`** | `Interface.Registry`, `ConversationRegistry`, `PubSub` |
| **`Processor`** | `Inference`, `Operations`, `Recipes` |

## 5. ETS Tables Summary

| Table Name | Owner | Purpose |
| :--- | :--- | :--- |
| `:"#{name}_conversations"` | `ConversationRegistry` | Active conversation state and context lookups. |
| `:"#{name}_cache"` | `Cache.Manager` | SHA-indexed LLM context layers (Haiku/Sonnet/Opus). |
| `:entity_cache` | `MemoryStore` | Read-optimized mirror of the `entities` table. |
| `:hot_topics` | `MemoryStore` | Decaying register of salient terms. |

## 6. SQLite Tables Summary

### `myelin.db` (MemoryStore)
- `entities`: Identity metadata, tiers, and salience.
- `rules`: Multiplicative salience and behavioral rules.
- `briefings`: Stored research results and conversation summaries.
- `conversations`: Long-term persistence of thread state.

### `knowledge.db` (KnowledgeStore)
- `knowledge`: Document content and metadata.
- `knowledge_fts`: FTS5 index for keyword search.
- `knowledge_vecs`: Vector embeddings for semantic similarity.

## 7. Type System Overview

- **`%Myelin.Inference.Request{}`**: Canonical messages-format request for LLM tasks.
- **`%Myelin.Inference.Response{}`**: Normalized response from any inference backend.
- **`%Myelin.Inference.Receipt{}`**: Detailed execution metadata (tokens, cost, latency) for audit and budgeting.
- **`%Myelin.Event{}`**: The universal ingestion packet. Contains `source`, `kind`, `payload`, `thread_id`, and `signals`.
- **`%Myelin.OutputEvent{}`**: The universal delivery packet. Contains `action`, `target`, `content`, and `status`.
- **`%Myelin.Conversation{}`**: The heavy state object for multi-turn history.
- **`%Myelin.ConversationContext{}`**: A subset of conversation state used for rapid enrichment.
- **`%Myelin.Interface.Capabilities{}`**: Defines what an interface can do (actions, max_length, etc.).
- **`%Myelin.StateMachine.Policy{}`**: Thresholds for attention (event_floor, escalation_threshold).
