# Backend Configuration

Myelin uses a pluggable backend system that allows swapping between local SLMs (Small Language Models) and remote LLM providers (like Anthropic). Each backend is defined by a TOML configuration file.

## 1. Overview

The backend configuration system allows the agent to:
- Declare model capabilities (chat, reasoning, routing, etc.)
- Define connection details and calling conventions
- Specify model-specific limits (context window, max output)
- Track costs and performance metrics for intelligent routing
- Configure provider-specific features like prompt caching

## 2. File Location

Configuration files are located in `config/backends/`.
- **Active Backends**: All `*.toml` files in this directory are loaded at startup.
- **Templates**: `.example.toml` files serve as templates and are ignored by the loader unless renamed to `.toml`.

## 3. Required Fields

Every backend configuration must include the following fields:

| Field | Type | Description |
| :--- | :--- | :--- |
| `name` | String | Unique identifier for the backend (e.g., "toybox", "anthropic-sonnet"). |
| `model` | String | The model ID passed to the provider's API (e.g., "claude-3-5-sonnet-20240620"). |
| `endpoint` | String | Base URL for the API. |
| `calling_convention` | String | The API format: `openai`, `anthropic`, or `llama` (for llama.cpp native). |
| `capabilities` | List | List of capabilities the model supports (see [Capabilities](#5-capabilities)). |
| `context_window` | Integer | Maximum input tokens the model can handle. |
| `max_output_tokens` | Integer | Maximum tokens the model can generate in a single response. |

## 4. Optional Sections

### `api_key_env`
The name of the environment variable (resolved via [Secrets Management](./07-secrets.md)) containing the API key.
```toml
api_key_env = "ANTHROPIC_API_KEY"
```

### `cost`
Defines pricing per million tokens. Used by the `Budget` system for tracking and `InferenceRouter` for economic routing.
```toml
[cost]
input_per_mtok = 3.00
output_per_mtok = 15.00
cache_write_per_mtok = 3.75
cache_read_per_mtok = 0.30
```

### `cache`
Configuration for prompt caching.
- `marker_style`: `:anthropic`, `:openai`, or `:none`.
- `ttl_tiers`: List of named TTL buckets (seconds) for cached layers.
```toml
[cache]
marker_style = "anthropic"
ttl_tiers = [
  { name = "ephemeral", seconds = 300 },
  { name = "standard", seconds = 3600 }
]
```

### `performance`
Performance metrics required for backends with the `routing` capability.
- `max_ttft_ms`: Maximum expected Time To First Token.
- `min_tok_per_sec`: Minimum expected throughput.
- `max_p99_ms`: Maximum P99 latency for full completion.

### `structured_output`
Specifies how the model handles constrained output: `gbnf` (for llama.cpp) or `json_schema`.

## 5. Capabilities

Capabilities define what tasks a backend can be assigned.

### Primitives
- `chat`: General conversation.
- `reasoning`: Complex analysis and tool use (Sonnet-tier).
- `routing`: Fast, low-latency evaluation (2B-tier).
- `summarization`: Efficient context compression.
- `tool_use`: Support for structured tool calling.
- `embedding`: Vector generation.
- `structured_output`: Can follow GBNF or JSON Schema strictly.

### Composites
- `agent`: Shorthand for `chat`, `reasoning`, and `tool_use`.

## 6. Degraded Mode

If a backend is missing its required `api_key_env` or fails health checks, it enters **Degraded Mode**.
- The system will log a warning but will **not crash**.
- The backend is marked as `:offline` or `:degraded` in the `BackendPool`.
- Tasks requiring that backend's unique capabilities will trigger a `backend_critical` alert via PubSub.

## 7. Adding a New Backend

1.  **Create Config**: Copy `config/backends/anthropic-sonnet.example.toml` to `config/backends/my-new-backend.toml`.
2.  **Fill Fields**: Update the `name`, `model`, `endpoint`, and `capabilities`.
3.  **Add Secret**: If it requires an API key, add the key to your `.secrets` file under the name specified in `api_key_env`.
4.  **Restart/Reload**: Restart the Myelin node. (Note: Hot-reloading of backend configs is planned but not yet implemented).

## 8. Examples

### Local Llama (via llama-server)
```toml
name = "toybox"
model = "qwen2.5-7b"
endpoint = "http://toybox:8083"
calling_convention = "llama"
capabilities = ["routing", "structured_output"]
structured_output = "gbnf"
context_window = 8192
max_output_tokens = 2048

[performance]
max_ttft_ms = 750
min_tok_per_sec = 5
max_p99_ms = 12000
```

### Remote Anthropic
```toml
name = "anthropic-sonnet"
model = "claude-3-5-sonnet-20240620"
endpoint = "https://api.anthropic.com/v1"
api_key_env = "ANTHROPIC_API_KEY"
calling_convention = "anthropic"
capabilities = ["agent", "reasoning"]
context_window = 200000
max_output_tokens = 8192
```

## 9. Startup Loading

The `Backend.Loader` module is responsible for the automated discovery and registration of backends during system boot.

- **Auto-Discovery**: At startup, the system scans `config/backends/` for all `*.toml` files.
- **Registration**: Each valid configuration is converted into a `BackendPool` entry.
- **Operability Checks**: The loader verifies that required secrets (defined in `api_key_env`) are available via the `Secrets` system.
- **Offline Status**: If a backend is missing its required API key, it is registered with an `:offline` status. The system continues to run, but that backend will not be used for tasks.
- **System Requirements**: For the agent to be functional, the `BackendPool` must contain at least one operable backend providing the following capabilities:
  - `:routing`: For initial event evaluation and vibe checks.
  - `:processing`: For complex reasoning, tool use, and response generation.
- **Validation Logs**: The loader logs a summary of loaded backends and issues `Logger.warning` messages if any required capability coverage is missing.
