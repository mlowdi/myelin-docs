# Event System

The Event System is the core ingestion and processing funnel for all external and internal signals in Myelin. It transforms raw data into enriched, scored events that drive the autonomous behavior of the agent.

## 1. Event Struct

The `%Myelin.Event{}` struct is the primary data carrier throughout the system.

### Fields and Types

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `String.t()` | Unique identifier (ULID). Generated on creation. |
| `source` | `atom()` | The source of the event (e.g., `:bluesky`, `:telegram`, `:syslog`, `:timer`, `:internal`). |
| `kind` | `atom()` | The specific type of event (e.g., `:mention`, `:reply`, `:message`, `:sigma_hit`, `:system`, `:timer`). |
| `payload` | `map()` | The parsed content of the event (e.g., `%{"text" => "..."}`). |
| `sender` | `String.t() \| nil` | The identity of the sender (e.g., handle or hostname). |
| `thread_id` | `String.t() \| nil` | The conversation identifier. Maps to platform-specific thread IDs or URIs. |
| `platform` | `atom()` | The platform where the event originated (e.g., `:bluesky`, `:telegram`, `:system`). |
| `raw` | `map()` | The raw, unparsed data from the source for debugging and archival. |
| `signals` | `map()` | Enriched data added during processing (e.g., salience score, entity tier, keyword hits). |
| `inserted_at` | `DateTime.t()` | UTC timestamp when the event was created. |

## 2. Event Lifecycle

An event flows through the following stages:

1.  **Creation**: An interface constructs an `%Event{}` struct.
2.  **Ingestion**: The event is passed to `EventPipeline.ingest/2`.
3.  **Filtering**: `InputFilter` checks the payload for blocked patterns (e.g., canary strings).
4.  **Enrichment**: `EventEnricher` fetches metadata (entities, keywords, hot topics, conversation context) in parallel.
5.  **Initial Scoring**: `Salience.score/1` computes a base salience score.
6.  **Queueing**: The event is added to the `EventPipeline` queue. If the queue is full, the lowest salience event is dropped (respecting protected conversations).
7.  **Routing**: The event is emitted to `InferenceRouter`.
8.  **Rule Application**: `InferenceRouter` applies active multiplicative rules and computes the final salience.
9.  **Inference**:
    *   **Pass 1 (Vibe Check)**: Determines if the event is `:notable`, `:urgent`, or `:quiet`.
    *   **Pass 2 (Action Decision)**: If notable, determines the specific action (e.g., `:react`, `:watch`, `:ignore`).
10. **Decision**: The final decision is broadcast via PubSub and triggers state evaluation in the `StateMachine`.

## 3. Sources

The following modules construct `%Event{}` structs and ingest them into the system:

*   **`Myelin.Interface.Bluesky` / `Myelin.Interface.Telegram`**: Construct events from platform-specific messages and mentions. Set `platform`, `sender`, `thread_id`, and `payload`.
*   **`Myelin.Interface.Syslog`**: Converts system log entries into events. Sets `source: :syslog`, `kind: :log_event` or `:sigma_hit`.
*   **`Myelin.Timer`**: Generates `:timer` events based on scheduled intervals.
*   **`Myelin.Interface.Internal`**: Emits system-level events like `:state_changed`, `:cache_available`, and `:budget_alert`.
*   **`Myelin.TaskScheduler`**: Generates `:deferred_task` events when scheduled tasks fire.

## 4. Sinks

Events eventually reach these terminal points or triggers:

*   **`Myelin.InferenceRouter`**: The primary consumer that performs LLM-based routing decisions.
*   **`Myelin.StateMachine`**: Evaluates events to potentially transition the agent's behavioral state.
*   **PubSub ("routing" topic)**: Decisions are broadcast to any subscribers (e.g., UI monitors, logging sinks).
*   **`Myelin.ConversationRegistry`**: Updates conversation state and tracking levels based on event activity.

## 5. EventPipeline

The `EventPipeline` is a `GenServer` that acts as the central ingestion funnel.

*   **State**: Manages a `:queue` of events and tracking stats (ingested, dropped, processed).
*   **Back-pressure**: When the queue exceeds `max_queue_size` (default 1000), it drops the event with the lowest salience. Events in `:watched`, `:active`, or `:interactive` conversations are prioritized for retention.
*   **Sequential Processing**: Events are processed one-by-one using `handle_continue` to ensure enrichment and scoring happen in order without blocking the ingestion of new events.

## 6. EventEnricher

The `EventEnricher` adds context to events by performing four parallel `Task` lookups:

1.  **Entity Lookup**: Checks `MemoryStore` for the sender's tier and salience.
2.  **Keyword Matching**: Matches event text against global keywords in `MemoryStore`.
3.  **Hot Topic Scoring**: Compares text against current "hot topics" for relevance.
4.  **Conversation Context**: Retrieves or creates a `ConversationContext` from the `ConversationRegistry` using the `thread_id`.

## 7. InputFilter

The `InputFilter` is a security and safety layer that runs before enrichment. It scans the event payload for:
*   **EICAR test strings**: Standard antivirus test patterns.
*   **LLM Canaries**: (e.g., "anthropic-canary-test-string") to prevent accidental leakage of prompt instructions or "jailbreak" attempts embedded in external content.

## 8. Salience Scoring

Salience determines how "interesting" an event is to the agent.

### Score Components (Base)
*   **Sender Tier**: Tier 1 (0.6), Tier 2 (0.3), Tier 3 (0.1), Others (0.05).
*   **Hot Topic Match**: Multiplier of 0.3 on the hot topic score.
*   **Entity Salience**: Multiplier of 0.2 on the entity's own salience.
*   **Keyword Hits**: 0.15 per hit (capped at 0.4).
*   **Conversation Boost**: Up to 0.4 based on tracking level (`:interactive` gets +0.3) and salience trend.

### Kind Multipliers
*   `:mention`: 1.5x
*   `:sigma_hit`: 1.8x
*   `:message`: 1.4x
*   `:reply`: 1.3x
*   `:like`: 0.3x (de-prioritized)

### Rules Applicator
Multiplicative rules (stored in `MemoryStore`) can be applied to the raw score. Rules include decay functions (`:linear`, `:exponential`) that move the multiplier back toward 1.0 (neutral) over time.

## 9. Dependencies

The event system depends on the following subsystems:
*   **`MemoryStore`**: For entity data, keywords, hot topics, and active rules.
*   **`ConversationRegistry`**: For tracking-level state and thread history.
*   **`InferenceRouter`**: For the final routing of enriched events.
*   **`LlamaClient`**: (Indirectly via Router) for vibe checks and action decisions.
*   **`Ulid`**: For unique ID generation.
*   **`Jason`**: For JSON parsing in interfaces.

## 10. ETS Tables

*   **`:"#{registry_name}_conversations"`**: Owned by `ConversationRegistry`. Stores `Conversation` and `ConversationContext` for hot-path access during enrichment.
*   **`:"#{name}_cache"`**: Owned by `Cache.Manager`. Used for caching LLM responses and intermediate pipeline results.

## 11. Types and Protocols

*   **`Myelin.Event.t()`**: The core type specification for events.
*   **`Myelin.Interface` behaviour**: Defines the contract for modules that create events (e.g., `capabilities/0`, `deliver/1`, `can_deliver?/1`).
*   **`Salience` logic**: While not a formal protocol, it follows a functional transformation pattern `Event -> Signals -> Score`.
