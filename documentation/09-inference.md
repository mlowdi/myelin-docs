# Inference Unification Layer

The Inference layer provides a unified, capability-based abstraction for interacting with Large Language Models (LLMs). It serves as the single entry point for all inference tasks within Myelin, ensuring that "Myelin speaks Myelin" internally while translating to various backend wire formats.

## Why it Exists

Before the unification layer, Myelin components (like `InferenceRouter`, `Engaged`, and `Processor`) interacted directly with leaf clients like `LlamaClient` or `AnthropicClient`. This led to several issues:
- **Tight Coupling**: Components were tied to specific model APIs.
- **Inconsistent Recording**: Usage and cost tracking were implemented differently across clients.
- **Capability Gaps**: Switching between local and remote models required manual code changes in the callers.

The Inference layer abstracts these concerns by allowing callers to request **capabilities** (e.g., `:routing`, `:reasoning`) and **locality constraints** (e.g., `:local`, `:remote`) rather than specific backends.

## The API: `Inference.complete/2`

The primary interface for the inference layer is `Myelin.Inference.complete/2`. All high-level components in Myelin have been migrated to use this unified entry point.

```elixir
{:ok, response, receipt} = Myelin.Inference.complete(request)
```

- **`request`**: A `%Myelin.Inference.Request{}` struct defining the task.
- **`response`**: A `%Myelin.Inference.Response{}` struct containing the normalized result.
- **`receipt`**: A `%Myelin.Inference.Receipt{}` struct containing execution metadata (tokens, cost, latency).

The system handles backend resolution via `BackendPool`, message translation, execution, and budget recording automatically.

## Dependency Injection: `inference_mod`

To support testability and modularity, all callers accept an optional `:inference_mod` in their configuration or `start_link` options. This defaults to `Myelin.Inference` in production but allows tests to inject mocks or specialized inference handlers.

Components utilizing this pattern include:
- `InferenceRouter`
- `Processor` (and `Recipes`)
- `Attentive`, `Engaged`, and `Interactive` layers
- `ResearchThread`
- `WebFetch`

## Request Struct (`Myelin.Inference.Request`)

The `Request` struct uses a canonical messages-only format. Raw prompts are wrapped into user messages internally by translators if the backend requires a flat prompt.

| Field | Type | Description |
| :--- | :--- | :--- |
| `:capability` | `atom` | Required capability (`:routing`, `:processing`, `:reasoning`, `:interactive`, `:batch`). |
| `:purpose` | `string` | Human-readable label for the task (e.g., `"vibe_check"`, `"engaged_reasoning"`). |
| `:messages` | `list` | Canonical messages: `[%{role: "system" | "user" | "assistant", content: "..."}]`. |
| `:options` | `map` | Model-specific options: `max_tokens`, `temperature`, `tools`, `grammar`, etc. |
| `:locality` | `atom | list` | Constraints: `:local`, `:remote`, `:sovereign`, or a list of these. |
| `:urgency` | `atom` | Priority of the request (`:normal`, `:high`). |
| `:session_id` | `string` | Optional ULID for session-based attribution and budget tracking. |

## Response Struct (`Myelin.Inference.Response`)

All backends return a normalized response, regardless of their native wire format.

| Field | Type | Description |
| :--- | :--- | :--- |
| `:content` | `string` | The generated text response. |
| `:tool_calls` | `list` | Normalized tool calls: `[%{id: "...", name: "...", input: %{...}}]`. |
| `:stop_reason` | `atom` | Why the generation stopped (`:end_turn`, `:tool_use`, `:max_tokens`, `:unknown`). |
| `:raw` | `any` | The original response from the leaf client (retained for debugging). |

## Receipt Pattern (`Myelin.Inference.Receipt`)

Every successful call returns a `Receipt`. This is a stable value object that provides an audit trail and feedback loop for the system.

| Field | Type | Description |
| :--- | :--- | :--- |
| `:request_id` | `string` | Unique ULID for this specific request. |
| `:backend_id` | `atom` | ID of the backend that handled the request (e.g. `:toybox`). |
| `:locality` | `atom` | Locality of the executing backend (`:local`, `:remote`, `:sovereign`). |
| `:capability` | `atom` | The capability used for this request. |
| `:purpose` | `string` | The purpose label from the request. |
| `:tokens` | `map` | Token usage: `%{input, output, cache_read, cache_write}`. |
| `:cost` | `float` | Estimated cost of the call in USD. |
| `:latency_ms` | `integer` | Total time taken for the call in milliseconds. |
| `:timestamp` | `DateTime` | When the request was completed. |

## Budget & Recording

Inference recording is centralized in `Myelin.Inference.complete/2`. It automatically extracts token counts and records them to `Myelin.Budget` using the appropriate tier (`:local`, `:haiku`, `:sonnet`, `:opus`).

- **Centralization**: Leaf clients no longer handle their own budget recording (as of Phase 4), ensuring consistent tracking across all backends.
- **Session Attribution**: If a `session_id` is provided in the `Request`, the usage is linked to that specific conversation session.

## Locality & Backend Resolution

Backends declare their `locality` in their configuration. Callers use the `locality` field in the `Request` to enforce data residency or performance tradeoffs:

- **`:local`**: Machine-local inference (e.g., local llama.cpp). Lowest latency, highest privacy, zero cost.
- **`:remote`**: External API providers (e.g., Anthropic, OpenRouter). Highest capability, variable cost.
- **`:sovereign`**: Self-hosted but not machine-local (e.g., an on-prem GPU farm). High capability, high privacy.

If a request specifies `locality: :local`, the `BackendPool` will only consider healthy backends marked as local.

## Calling Conventions

The system supports translation for the following conventions via `Myelin.Inference.Translator`:
- **`:llama`**: Flattens messages to a prompt string using basic templates; supports GBNF grammars. Used for local `llama.cpp` backends.
- **`:anthropic`**: Maps roles to the Anthropic Messages API; supports native tool use and prompt caching.
- **`:openai`**: (Stub) Passes messages directly; intended for OpenAI-compatible APIs.

## Minimum Viable Deployment

The system is designed to be operational with minimal resources. A valid deployment requires at least **one healthy model** that declares both `:routing` and `:processing` capabilities, at any locality. This can be a single local `llama.cpp` instance or a single remote API key.

## Migration Status

The Inference unification is complete (Phases 1-4):

- **Phase 1**: Foundation implementation (Request/Receipt/Response structs, Inference supervisor, Translators).
- **Phase 2**: Migration of `InferenceRouter` and `Processor`.
- **Phase 3**: Migration of `Attentive`, `Engaged`, `Interactive`, `ResearchThread`, and `WebFetch`.
- **Phase 4**: Centralized budget recording and removal of direct leaf client usage in higher layers.

*Status: Migration Complete. All internal components communicate via the unified Inference layer.*
