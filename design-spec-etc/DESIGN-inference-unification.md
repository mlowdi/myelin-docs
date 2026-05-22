# Design Spec: Unified Inference Abstraction

## Context & Motivation

Currently, the Myelin codebase interacts with LLMs through distinct leaf clients: `LlamaClient` and `AnthropicClient`. This leads to tight coupling, inconsistent usage recording, and capability gaps when switching between local and remote models.

This design introduces `Myelin.Inference`, a unified abstraction layer. Callers request capabilities and locality constraints rather than specific backends. The inference layer is an interface, analogous to platform interfaces like Telegram or Bluesky: Myelin speaks a canonical internal format (messages) everywhere, and the inference layer translates it to the wire format of the chosen backend.

## Proposed Architecture

The `Inference` module sits above the leaf clients, using `BackendPool` to resolve the best backend for a requested capability and locality.

```text
[ Callers ] (Engaged, Interactive, Router, Processor)
      │
      ▼
[ Myelin.Inference ] ──► [ Budget ] (Automatic usage recording)
      │
      ├───────────────────┐
      ▼                   ▼
[ BackendPool ]     [ Translator ] (Messages → Wire Format)
      │                   │
      ▼                   ▼
[ LlamaClient ]     [ AnthropicClient ]    [ OpenAIClient ]
```

### Key Modules
- `Myelin.Inference`: Orchestrates backend resolution, translation, execution, and budget recording. Returns a receipt to the caller.
- `Myelin.Inference.Request`: The canonical messages-format request.
- `Myelin.Inference.Receipt`: Value object detailing execution metadata.
- `Myelin.Inference.Response`: Normalized response format.
- `Myelin.Inference.Translator`: Converts canonical messages into client-specific payloads.

## Locality Constraints

Backends declare `locality` as a routing constraint, not a capability. This allows callers to enforce data residency or cost/latency tradeoffs.

- `:local` — Machine-local inference (e.g., local llama.cpp)
- `:remote` — External API (e.g., Anthropic, OpenRouter)
- `:sovereign` — Self-hosted but not machine-local (e.g., on-prem GPU farm)

TOML config addition:
```toml
locality = "local"  # or "remote" or "sovereign"
```

Callers filter by locality in `BackendPool`:
```elixir
BackendPool.route_task([:reasoning], locality: :local)
BackendPool.route_task([:processing, :batch], locality: [:local, :sovereign])
```

## API Surface

### Canonical Request Format

The canonical internal format is strictly messages-based. Raw prompt strings are wrapped into user messages. There are no dual message/prompt fields.

```elixir
defmodule Myelin.Inference.Request do
  defstruct [
    :capability,    # :routing | :processing | :reasoning | :interactive | :batch
    :purpose,       # string (for receipt annotation, e.g. "engaged_reasoning", "vibe_check")
    :messages,      # [%{role: "system" | "user" | "assistant", content: binary()}]
    :options,       # %{max_tokens: int, temperature: float, tools: list, grammar: binary, ...}
    :urgency,       # :normal | :high (default :normal)
    :locality,      # :local | :remote | :sovereign | [:local, :sovereign] | nil (nil = any)
    :session_id     # session_id for attribution (nil for background/maintenance work)
  ]
end
```

### The Inference Receipt Pattern

`Inference.complete/2` returns a receipt alongside the response. The Inference layer handles Budget recording automatically. The caller decides how to persist or link the receipt (e.g., passing it to `StateMachine` or a session). Sessions store `[receipt.request_id, ...]` as references, keeping the session schema stable.

```elixir
defmodule Myelin.Inference.Receipt do
  defstruct [
    :request_id,    # ulid
    :backend_id,    # atom (e.g., :cortex)
    :locality,      # :local | :remote | :sovereign
    :capability,    # atom
    :tokens,        # %{input: int, output: int, cache_read: int, cache_write: int}
    :cost,          # float
    :latency_ms     # integer
  ]
end
```

Example usage:
```elixir
{:ok, response, receipt} = Myelin.Inference.complete(req)
# Caller stores receipt.request_id in session
```

## Calling Convention Translation

The `Translator` converts canonical messages to the backend's `calling_convention`. All conventions return a normalized `Inference.Response`.

| Convention | Input Translation | Output Translation |
|------------|-------------------|-------------------|
| `:llama` | Flatten messages to prompt string via chat template. Extract `grammar` from options. | Wrap string in `%{content: string}` |
| `:anthropic` | Extract system message, map roles. Pass `tools`. | Extract `text` or `tool_use` from response. |
| `:openai` | Pass messages directly. Map `tools` to OpenAI format. | Extract `choices[0].message`. |

## Minimum Viable Deployment

The system requires only 1 model with `:routing` + `:processing` capabilities, at any locality. It works on a single local llama.cpp instance or a single OpenRouter key. Any backend declaring `:routing` can route; there is no local-first constraint for routers. Everything else is additive.

## Migration Plan

1. **Phase 1: Foundation**
   - Add `locality` field to `Backend.Config` and TOML parsing.
   - Add `locality` filter to `BackendPool.route_task/2`.
   - Implement `Inference.Receipt` struct.
   - Implement `Inference.Request` struct with messages-only format.
   - Implement `Myelin.Inference` and translators.
   - Update `Budget` to accept usage from the inference layer.

2. **Phase 2: Local Core**
   - Migrate `InferenceRouter`.
   - Migrate `Processor`.

3. **Phase 3: Remote Layers**
   - Migrate `Attentive`, `Engaged`, and `Interactive`.

4. **Phase 4: Cleanup**
   - Remove manual usage recording from `AnthropicClient`.
   - Ensure all callsites use the unified layer.

## Open Questions
- Should the unified layer support streaming responses natively for `Interactive` sessions?
- Does `BackendPool` need dynamic health-based rerouting mid-request if a selected backend fails during `Inference.complete`?
