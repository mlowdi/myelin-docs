# Secrets Management

The `Myelin.Secrets` system provides centralized, secure management of sensitive material such as API keys and platform tokens. It is designed to ensure agent isolation, support hot-reloading of credentials, and provide a single audit point for all secret access.

## 1. Overview

Secrets are managed by a dedicated GenServer that maintains material in **process state only**. By avoiding ETS or other global shared memory for secret storage, Myelin reduces the risk of accidental exposure through memory dumps or logging of global state.

The system is a **Tier 1 leaf service**, meaning it is started early in the `Myelin.Application` lifecycle before any services that might require credentials (like Interfaces or Inference Backends).

## 2. Secret Sources

Secrets are loaded at startup and can be merged from multiple sources, following this priority:

1.  **`.secrets` file** (Preferred): A local TOML-like file (standard `KEY=VALUE` format) that is not committed to version control.
2.  **`.env` file** (Fallback): Standard environment file used as a secondary source.
3.  **System Environment Variables**: The underlying system environment acts as the final fallback.

### File Format
Files should use the standard `KEY=VALUE` format:
```bash
# Platform Keys
TELEGRAM_BOT_TOKEN=123456:ABC-DEF
ANTHROPIC_API_KEY="sk-ant-..."

# Interface Config
BLUESKY_PASSWORD='my-safe-password'
```
- Supports `#` for comments.
- Supports both single (`'`) and double (`"`) quotes for values.
- Leading/trailing whitespace is trimmed.

## 3. API Reference

All calls are routed to the `Myelin.Secrets` GenServer.

### `Secrets.get(key)`
Returns the value associated with the key or `nil` if not found.
```elixir
api_key = Myelin.Secrets.get("ANTHROPIC_API_KEY")
```

### `Secrets.available?(key)`
Returns a boolean indicating if the key exists in the current state.
```elixir
if Myelin.Secrets.available?("OPENAI_API_KEY") do
  # ...
end
```

### `Secrets.check_requirements(keys)`
Validates that a list of required keys are all present. Returns `{:ok, :satisfied}` or `{:error, missing_keys}`.
```elixir
case Myelin.Secrets.check_requirements(["BSKY_HANDLE", "BSKY_PASSWORD"]) do
  {:ok, :satisfied} -> :ok
  {:error, missing} -> Logger.error("Missing credentials: #{inspect(missing)}")
end
```

### `Secrets.put(key, value)`
Injects a secret into the running process state. Primarily used for testing or dynamic runtime injection. Does not persist to disk.

### `Secrets.reload()`
Triggers a fresh read of the `.secrets` (or `.env`) file and merges it into the current state. This allows for **hot-rotation of keys** without restarting the entire Myelin node.

## 4. Integration Points

The following core components rely on the Secrets system:

- **`AnthropicClient`**: Retrieves `ANTHROPIC_API_KEY` for inference.
- **`Interface.Supervisor`**: Retrieves platform-specific tokens (Telegram, Bluesky) when spawning interface workers.
- **`Backend.ConfigLoader`**: Validates that required `api_key_env` variables for backend configurations are present in the environment.

## 5. Security Model

- **Process Isolation**: Secrets live only in the `Myelin.Secrets` process heap.
- **No Propagation**: Secrets MUST NOT be included in `%Myelin.Event{}` structs, logged to disk, or surfaced in LLM context windows.
- **Audit Point**: All access to secrets flows through `Secrets.get/1`, providing a single point where access logging or rate limiting could be implemented.
- **Pluggability**: The internal `load_secrets` logic can be extended to support external vaults (HashiCorp Vault, AWS KMS) without changing the public API.

## 6. Testing & Operations

### Testing
To test modules that depend on secrets without setting real environment variables, use the `:name` option to start a local instance or use `Secrets.put/2`:
```elixir
# In a test setup
Myelin.Secrets.put("MOCK_KEY", "mock-value")
```

### Key Rotation
To rotate a key in production:
1. Update the `.secrets` file with the new value.
2. Call `Myelin.Secrets.reload()` via the IEx console or an administrative trigger.
3. Subsequent calls to `Secrets.get/1` will return the new value.
