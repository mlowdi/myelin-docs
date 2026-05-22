# Interfaces & Output System

This document describes the Myelin interface layer (input/output adapters) and the delivery pipeline for outbound actions.

## 1. Interface Registry

`Myelin.Interface.Registry` is a GenServer that maintains a mapping of active interface modules to their `Capabilities`. 

- **Registration**: Interfaces call `Registry.register(__MODULE__)` during their initialization (typically via the application supervisor).
- **Metadata**: Stores `Myelin.Interface.Capabilities` structs for each interface.
- **Context Rendering**: Provides `render_context_block/0`, which uses `Myelin.Interface.ContextRenderer` to generate a formatted text block of all available actions, identities, and constraints. This block is injected into the LLM context to inform the agent of its available "body parts" on various platforms.
- **Routing**: Used by the `OutputPipeline` to verify if a target interface exists and if it can handle a specific action.

## 2. Telegram Interface

`Myelin.Interface.Telegram` provides bidirectional communication with the Telegram Bot API.

- **Polling**: Uses a GenServer-based polling loop with `getUpdates`. It implements exponential backoff on errors and respects `retry_after` headers from Telegram's rate limiter (429).
- **Event Creation**: Maps Telegram `message` objects to `Myelin.Event` structs. `chat_id` is mapped to `thread_id` to maintain conversation locality.
- **Registration**: Implements a challenge-response mechanism. A user must initiate registration via a human-out-of-band process (e.g., getting a code from the CLI) and then message the bot with `register <code>`. This binds the interface to a specific Telegram user ID, preventing unauthorized interaction.
- **Public API**: Includes `get_me/0`, `begin_registration/0`, `registered_user/0`, and `clear_registration/0`.
- **Delivery**: Uses `Req` to POST to Telegram's `/sendMessage`, `/editMessageText`, and `/sendPhoto` endpoints.

## 3. Bluesky Interface

`Myelin.Interface.Bluesky` wraps a TypeScript-based bridge process to interact with the AT Protocol.

- **TypeScript Bridge**: A Node.js process (`priv/bluesky/bridge.ts`) that uses the `@atproto/api` and `@atproto/identity` libraries. It maintains the session and handles the complex lexicon-based signing/posting.
- **Communication**: Uses an Elixir `Port`. Commands are sent as JSON-lines to `stdin`. Events (mentions, replies, likes) are received as JSON-lines from `stdout`.
- **Event Creation**: Maps ATProto events to `Myelin.Event` structs. `uri` or `thread_id` is used to track the conversation.
- **ATProto Adapter**: The bridge handles the translation between internal JSON commands and the specific XRPC calls required by Bluesky.

## 4. Syslog Interface

`Myelin.Interface.Syslog` is an **input-only** interface designed for system monitoring and observability.

- **Ingestion**: Accepts raw syslog strings via `ingest/2`.
- **Sigma Engine**: Each log entry is passed through `Myelin.Interface.Syslog.SigmaEngine`, which matches the entry against a set of security/operational rules (Sigma format).
- **Event Creation**: 
    1. A primary `log_event` is always created.
    2. If Sigma rules match, a secondary `sigma_hit` event is created, referencing the primary event ID. This allows the agent to react specifically to high-salience security or system alerts.

## 5. Interface Contract

Interfaces MUST implement the `Myelin.Interface` behaviour.

### Required Callbacks
- `capabilities()`: Returns a `%Myelin.Interface.Capabilities{}` struct defining supported actions, constraints, and platform metadata.
- `deliver(OutputEvent)`: Executes the outbound action. Returns `{:ok, receipt}` or `{:error, reason}`.
- `can_deliver?(OutputEvent)`: Predicate to check if the interface is capable of fulfilling a specific request (e.g., checking if an action is supported or if required target fields are present).

### Event Requirements
Interfaces are responsible for setting the following fields on ingested `Myelin.Event` structs:
- `source`: Atom identifier (e.g., `:telegram`).
- `kind`: Event type (e.g., `:message`, `:mention`, `:log_event`).
- `thread_id`: A string uniquely identifying the conversation context (crucial for state retrieval).
- `platform`: The platform atom (e.g., `:telegram`, `:bluesky`).
- `payload`: Map containing the primary content (e.g., `text`).

## 6. Output Pipeline

`Myelin.OutputPipeline` processes all outbound `OutputEvent` structs through an 8-stage lifecycle.

1.  **Validate**: Ensures the event has a valid action, content, and target interface.
2.  **Constrain**: Checks the content against interface-defined constraints (e.g., `max_length`) and verifies that required `target` fields (like `chat_id`) are present.
3.  **Prioritize**: Assigns or validates the priority level (`:low`, `:normal`, `:urgent`).
4.  **Approve**: If `requires_approval` is true, the event is moved to a `pending_approval` state and held until a human approves it via `OutputPipeline.approve/1`.
5.  **Rate Limit**: Enforces a per-interface cooldown (default 1000ms). If the interface was used too recently, the event is queued.
6.  **Deliver**: Calls the interface module's `deliver/1` callback.
7.  **Receipt**: Updates the event status to `:delivered` and records the timestamp and platform-specific receipt (e.g., `message_id`).
8.  **Feedback**: 
    - Dispatches a `{:output_delivered, id, receipt}` message via PubSub (`"output_delivered"` topic).
    - Notifies `Myelin.ConversationRegistry` via `record_agent_action/2` to update the short-term memory of the conversation.

## 7. OutputEvent Struct

The `Myelin.OutputEvent` struct defines the schema for outbound actions.

| Field | Type | Description |
|---|---|---|
| `id` | `String (ULID)` | Unique identifier for the output. |
| `action` | `atom` | Action to perform (e.g., `:send_message`, `:like`). |
| `target_interface` | `module` | The interface module responsible for delivery. |
| `target` | `map` | Platform-specific addressing (e.g., `%{chat_id: 123}`). |
| `content` | `map` | The payload to deliver (e.g., `%{text: "Hello"}`). |
| `priority` | `atom` | `:low`, `:normal`, or `:urgent`. |
| `requires_approval` | `boolean` | If true, halts at Stage 4 for human review. |
| `status` | `atom` | Lifecycle state: `:pending`, `:pending_approval`, `:approved`, `:delivering`, `:delivered`, `:failed`. |
| `conversation_id` | `String` | Copied from `thread_id` for registry correlation. |
| `delivery_receipt` | `map` | Platform feedback (e.g., `message_id` or `error_reason`). |

## 8. Delivery Mechanics

Delivery is decoupled from the decision-making process.
1. The `StateMachine` or a Tool creates an `OutputEvent`.
2. It calls `OutputPipeline.submit(event)`.
3. The pipeline handles all orchestration (approval, queueing, rate-limiting).
4. The pipeline performs the final "hand-off" by calling `interface_module.deliver(event)`.

Platform adapters (Telegram, Bluesky) act as the terminal executors of these events, translating the generic `OutputEvent` into platform-specific HTTP requests or Port commands.

## 9. Dependencies

| Interface | External Dependencies | Internal Dependencies |
|---|---|---|
| **Telegram** | Telegram Bot API, `Req` | `EventPipeline`, `PubSub` |
| **Bluesky** | Node.js, `@atproto/api`, Port | `EventPipeline` |
| **Syslog** | Syslog protocol (inbound) | `SigmaEngine`, `EventPipeline` |
| **Internal** | Elixir PubSub | `EventPipeline`, `TaskScheduler` (planned) |
| **Registry** | n/a | `Capabilities`, `ContextRenderer` |

## 10. Formal Interface Behaviour

Myelin uses a **formal behaviour** defined in `lib/myelin/interface.ex`. 

The contract is explicit via `@callback` definitions. Any module that implements the `Myelin.Interface` behaviour and is registered with the `Interface.Registry` is automatically integrated into the system's routing and LLM context.

## 11. Extensibility

Adding a new interface **does not require modifying any core Myelin code** (except for optional convenience mapping in `Myelin.Interface.module_from_source/1`).

To add a new platform:
1. Create a new module (e.g., `Myelin.Interface.Discord`).
2. Implement the `Myelin.Interface` behaviour.
3. Start the module under the application supervisor (or as a transient child).
4. Call `Myelin.Interface.Registry.register(__MODULE__)` during `init/1`.
5. The interface's capabilities will automatically appear in the LLM's context block, and the `OutputPipeline` will be able to route events to it.
