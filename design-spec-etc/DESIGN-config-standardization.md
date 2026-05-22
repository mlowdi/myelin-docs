# Design: Configuration Standardization

*Draft 2026-03-29. Based on research in stuff/research/config-ingestion-map.md*

## Problem

Myelin has 8 config ingestion points using 4 formats (TOML, Elixir, KV, Markdown, JSON) with 3 different validation strategies (strict, basic, none) and 2 different loader architectures (parse-only vs parse-and-start). This happened organically and it's fine for now, but divergence will compound as we add more TOML-driven subsystems (knowledge providers, future memory backends, monitoring integrations).

The goal isn't to unify everything into one format — markdown personality files and JSON cursor state are what they are. The goal is to **standardize the TOML-driven loader pattern** that backends, interfaces, and knowledge providers share, and **explicitly allow the pattern to evolve** without creating silos.

## Current state

| Source | Format | Loader Pattern | Validation | Secrets |
|--------|--------|---------------|------------|---------|
| Backends | TOML | Parse → return structs (caller starts) | Strict (accumulated errors) | Warning on missing, still loads |
| Interfaces | TOML | Parse → start via DynamicSupervisor | Basic (skip on missing secret) | Skip entire interface |
| Knowledge | TOML | Parse → start via DynamicSupervisor | Basic (skip on missing secret) | Skip entire provider |

Three loaders doing the same job three different ways.

## Proposed: Unified TOML loader behaviour

### `Myelin.Config.Loader` behaviour

Extract the common pattern into a behaviour that all TOML-driven loaders implement:

```elixir
defmodule Myelin.Config.Loader do
  @doc "Scan directory, parse TOMLs, validate, return config structs."
  @callback load_dir(path :: String.t()) :: {:ok, [config()]} | {:error, [String.t()]}

  @doc "Parse a single TOML file."
  @callback load_file(path :: String.t()) :: {:ok, config()} | {:error, term()}

  @doc "Validate a parsed config. Returns accumulated errors."
  @callback validate(config :: map()) :: :ok | {:error, [String.t()]}

  @doc "Map a validated config to a child spec for DynamicSupervisor."
  @callback to_child_spec(config :: map()) :: Supervisor.child_spec()

  @doc "The directory to scan for TOML files."
  @callback config_dir() :: String.t()

  @doc "The DynamicSupervisor to start children under."
  @callback supervisor() :: atom()
end
```

A shared `Myelin.Config.Loader.start_all/1` function handles the common orchestration:

```elixir
def start_all(loader_module) do
  dir = loader_module.config_dir()
  supervisor = loader_module.supervisor()

  dir
  |> scan_toml_files()
  |> Enum.map(&loader_module.load_file/1)
  |> Enum.map(&loader_module.validate/1)
  |> Enum.each(fn
    {:ok, config} ->
      spec = loader_module.to_child_spec(config)
      DynamicSupervisor.start_child(supervisor, spec)
    {:error, reasons} ->
      Logger.warning("Config validation failed: #{inspect(reasons)}")
  end)
end
```

Each subsystem implements the behaviour with its own `validate/1` and `to_child_spec/1`. The strict validation from `Backend.ConfigLoader` becomes the model — accumulated errors, type checking, constraint verification.

### Standardized TOML shape

All TOML configs follow the same top-level structure:

```toml
# Required: identity
name = "my-thing"
type = "specific_type"

# Optional: metadata
tier = "capable"           # for backends
scope = "what I know"      # for knowledge providers
# ... subsystem-specific fields at root level

# Optional: secrets (always resolved via Myelin.Secrets)
[secrets]
api_key = "ENV_VAR_NAME"

# Optional: cost/performance/cache blocks (subsystem-specific)
[cost]
input = 0.10
output = 0.30
```

**Decided**: Options go at the root level, not in an `[options]` block. The Interface pattern wins over the Knowledge pattern — it's flatter and more readable. Knowledge.Loader gets updated to match.

### Standardized secret handling

One behavior for all loaders — **degraded start, not skip**:

1. Parse the `[secrets]` block
2. Resolve each via `Myelin.Secrets.get/1`
3. If ANY required secret is missing:
   - Log a clear warning: `"Config 'my-thing': missing secret FOO_KEY — starting in degraded mode"`
   - Still return the config, but with a `degraded: true` flag
   - The subsystem decides what "degraded" means (backend: can't make API calls, interface: can't authenticate, etc.)
4. If ALL secrets resolve: normal start

This is consistent with the backend pattern (warn + continue) but gives the subsystem explicit knowledge that it's degraded, rather than silently having a nil API key.

### Standardized validation

Port the `Backend.ConfigLoader` approach to all loaders:

```elixir
defp validate(config) do
  []
  |> check_required(config, [:name, :type])
  |> check_type_enum(config, :type, ~w(bluesky telegram miniflux))
  |> check_secrets(config)
  |> case do
    [] -> {:ok, config}
    errors -> {:error, errors}
  end
end
```

Accumulated errors, not fail-on-first. Every validation function appends to the error list. The caller gets ALL problems at once, not a game of whack-a-mole.

## What doesn't change

- **Elixir app config** (`config/config.exs`): stays as-is. It's compile-time constants, not runtime config.
- **Secrets file** (`.secrets`/`.env`): stays as-is. Simple KV is the right format for secrets.
- **Personality files** (`priv/*.md`): stays as-is. Markdown is the right format for LLM-facing text.
- **Manpages** (`priv/manpages/*.md`): stays as-is. Same reasoning.
- **Cursor/state files** (`.json`): stays as-is. These are runtime state, not config.

## Migration path — DONE (PRs #158, #159)

1. **Create `Myelin.Config.Loader` behaviour** with the common orchestration functions — DONE (#158)
2. **Refactor `Interface.Loader`** to implement the behaviour — DONE (#158)
3. **Refactor `Knowledge.Loader`** — flattened TOML shape, implements behaviour — DONE (#158)
4. **Refactor `Backend.ConfigLoader`** — implements behaviour, `to_child_spec/1` and `supervisor/0` made `@optional_callbacks` — DONE (#159)
5. **Update all `.example.toml` files** — standardized secrets block — DONE (#159)
6. **Add `Myelin.Config.Validator`** — shared validation helpers — DONE (#158)

## Future: Memory backend config

When we add TOML-driven memory backend configuration (currently hardcoded via `Application.get_env(:myelin, :memory_backend)`), it follows the same pattern:

```toml
# config/memory/engram.toml
name = "engram"
type = "engram"
scope = "Agent semantic memory"

[secrets]
engram_url = "ENGRAM_URL"

[options]
collections = ["memories"]
```

Loaded by `Myelin.Memory.Loader` implementing the same behaviour.

## Design principles

1. **TOML for declarative config, Elixir for programmatic config.** If a human edits it, TOML. If it's computed, Elixir.
2. **Flat over nested.** Options at root level. Only use TOML tables for semantically grouped blocks (`[secrets]`, `[cost]`, `[performance]`).
3. **Strict validation, graceful degradation.** Catch every error upfront, but don't refuse to start — start degraded and log clearly.
4. **Every TOML directory gets an `.example.toml`.** This IS the schema documentation.
5. **The pattern is allowed to evolve.** The behaviour is a convention, not a straitjacket. If a new subsystem needs something the behaviour doesn't provide, extend the behaviour — don't work around it.
