# Inference & Routing

The Inference and Routing subsystem orchestrates how Myelin perceives incoming events, determines their importance (salience), and decides whether to engage, ignore, or escalate them. It uses a multi-stage process involving local small language models (SLMs) for rapid routing and larger models for complex reasoning.

## 1. Two-pass Routing Flow
To minimize costs and latency while maintaining high accuracy, Myelin employs a two-pass routing strategy for all incoming events that are not explicitly bypassed.

- **Pass 1: Vibe Check (Classification)**
  - **Purpose**: Rapidly filter out noise and identify urgent or notable events.
  - **Model**: Local Llama (typically 2B-3B parameters).
  - **Output**: `:quiet` (drop), `:notable` (proceed to Pass 2), or `:urgent` (escalate immediately).
- **Pass 2: Action Decision (Routing)**
  - **Purpose**: Determine the specific system action for a notable event.
  - **Model**: Local Llama (typically 2B-3B parameters).
  - **Output**: `:drop`, `:watch` (track but don't respond), `:react` (enrich and potentially respond), or `:escalate` (hand off to a higher-tier model).

## 2. InferenceRouter
The `Myelin.InferenceRouter` GenServer is the central orchestrator for event routing.

- **Event Ingestion**: Receives events via `route_event/2` and processes them asynchronously using `Task` to avoid blocking the router.
- **Salience Computation**: Uses `Myelin.Salience` to score events against active rules retrieved from `MemoryStore`.
- **Routing Paths**:
  - **Policy Bypass**: If a conversation has the `bypass_router` policy override, it skips both passes and goes directly to `:react`.
  - **Interactive Bypass**: Events from conversations in the `:interactive` tracking level bypass the router to ensure low-latency human interaction. Sets outcome to `:react`.
  - **Standard Flow**: Follows the two-pass flow if salience meets the state's `event_floor`.
- **Speculative Tasks**: For high-salience events (≥ 0.7), the router triggers background tasks via `Myelin.Processor` (e.g., summarizing memories, compressing threads) to prepare context before a response is even decided.

## 3. StateMachine & Attention States
The `Myelin.StateMachine` manages the agent's internal attention level, which dictates how sensitive the router is to new events.

### The 5 Attention States
1.  **Dormant**: Minimum power mode. Only responds to high-salience triggers or known tier-1 senders.
2.  **Monitoring**: Passive observation. Tracking conversations but not yet actively participating.
3.  **Attentive**: High-alert. Actively evaluating if a response is needed (usually runs Haiku-tier inference).
4.  **Engaged**: Active participation. Formulating and delivering responses (usually runs Sonnet-tier inference).
5.  **Interactive**: Real-time human connection. Lowest thresholds and highest priority.

### State Policies
Each state defines a `Myelin.StateMachine.Policy` containing:
- `event_floor`: Minimum salience required to even consider the event.
- `escalation_threshold`: Salience required to trigger a state transition.
- `cooldown_seconds`: Minimum time to stay in state.

### Transitions & TTLs
- **Escalation**: Triggered by high salience, specific event kinds (e.g., `:human_connect`), or explicit model recommendations.
- **De-escalation**: Occurs via periodic checks (every 60s) based on silence timeouts or hard TTLs (e.g., 60 minutes for `:engaged`).

## 4. Inference Tiers & Routing Strategies
Myelin uses a 4-tier capability system to match tasks with appropriate model intelligence and cost profiles. A `%Myelin.Inference.Request{}` specifies a `min_tier`, and the routing system satisfies it based on configured strategies.

### The 4 Tiers
1.  **`:edge`**: On-device, hyper-fast, low capability (e.g., Llama 3 2B). Used for routing vibe checks and basic filtering.
2.  **`:efficient`**: Fast, cheap, capable of basic reasoning (e.g., Haiku, local 8B models). Used for the `Attentive` state, summarization, and simple memory tasks.
3.  **`:capable`**: Strong reasoning, moderate cost (e.g., Sonnet). Used for the `Engaged` state and complex data extraction.
4.  **`:frontier`**: Maximum intelligence, high cost (e.g., Opus). Used for complex, multi-step agentic planning or dense coding tasks.

### Routing Strategies
When resolving a request, the system uses a `strategy` (defaulting to `:prefer_cost`):
- `:prefer_cost`: Selects the cheapest backend that meets or exceeds the `min_tier`.
- `:prefer_speed`: Selects the fastest backend (lowest latency) that meets or exceeds the `min_tier`.
- `:prefer_quality`: Selects the highest available tier, ignoring cost constraints.

## 5. BackendPool & Health Management
`Myelin.BackendPool` manages a collection of compute resources (local or remote) that provide inference capabilities.

- **Registration**: Backends register with their endpoint, model name, `tier`, and `capabilities` (e.g., `[:router, :haiku, :embedding]`).
- **Health Checking**: The `HealthCheck` GenServer pings `/health` endpoints every 30s.
  - `3+` failures → `:offline`
  - `1-2` failures → `:degraded`
- **Tier Filtering**: Before routing, the pool filters out any backends that do not meet the request's `min_tier` or lack the required capabilities.
- **Routing**: `route_task/2` selects the best backend from the filtered pool based on the requested strategy, status, and historical latency. It supports urgency-aware selection (e.g., `:high` urgency can use `:degraded` backends).

## 6. Anthropic Client
The `Myelin.AnthropicClient` handles interaction with the Anthropic Messages API for high-tier inference (Haiku, Sonnet, Opus).

- **Prompt Caching**: Uses Anthropic's prompt caching by structuring system and message blocks. While the client extracts system messages, the `ContextBuilder` layers are designed to be stable for maximum cache hits.
- **Token Tracking**: Automatically records input, output, and cache (read/write) tokens for every request.
- **Cost Attribution**: Reports usage to the `Budget` GenServer immediately upon response.

## 7. ContextBuilder: The 4-Layer Pipeline
The `Myelin.ContextBuilder` assembles the complex prompts used by models, organized into four layers of stability to optimize prompt caching.

1.  **Layer 0 (Identity)**: The "Identity Bedrock." Contains the `Personality` definition. Cached for 1 hour.
2.  **Layer 1 (Situational)**: System-wide state including active rules, entity registry summaries, cost summaries, and interface capabilities.
3.  **Layer 2 (Session)**: Current state machine state, active policy, session notes, and hot topics.
4.  **Hot Zone (Per-request)**: The most volatile data. Includes the current event summary, relevant memory briefings, thread context, and specific instructions.

## 8. Budget & Cost Control
The `Myelin.Budget` GenServer tracks all external API spend and enforces safety limits.

- **Cost Recording**: Calculates USD cost based on pricing tiers for Haiku, Sonnet, and Opus.
- **Daily Budget**: Defaults to $0.50/day, resetting at 10:00 AM local time.
- **Alerting**: Broadcasts alerts via PubSub at 80% (warning) and 100% (critical) budget utilization.
- **Conservation Mode**: Can be manually or automatically triggered to restrict expensive inference calls for a set duration.

## 9. Dependencies & External Services
The inference subsystem depends on the following external infrastructure:
- **Router LLM**: Local `llama-server` (typically Qwen or Llama 3) for the two-pass routing flow.
- **Processor LLM**: Local or remote model for background enrichment tasks.
- **Anthropic API**: Cloud-based Claude models for complex reasoning and response generation.
- **Embeddings Service**: For vector search and memory retrieval.

## 10. ETS Tables & State Storage
- **MemoryStore (SQLite)**: Persists all rules, entities, and conversation history.
- **ConversationRegistry (ETS)**: Stores hot-path conversation state, including tracking levels and policy overrides.
- **Registry (Myelin.PubSub)**: Used for internal message broadcasting (routing decisions, state transitions, budget alerts).

## 11. Failure Handling & Resiliency
- **Inference Failure**:
  - If the Vibe Check fails, the router fallbacks to `:notable` (proceeding to the next stage).
  - If the Action Decision fails, it fallbacks to `:watch` (tracking but safe).
- **Backend Failover**: If a primary backend is offline, `BackendPool` automatically routes to the next best available provider with matching capabilities.
- **Rate Limiting**: The Anthropic client handles `429` responses by returning `{:error, :rate_limited}`, allowing calling modules to implement backoff strategies.

