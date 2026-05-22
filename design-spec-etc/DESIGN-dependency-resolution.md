# DESIGN: Declarative Dependency Resolution

**Status: Draft**

## 1. Problem Statement

Currently, validation of backend requirements is scattered across several modules:
- `ConfigLoader`: Validates TOML schema, secrets availability, and capability-specific performance bounds.
- `BackendPool`: Tracks runtime status (online/degraded/offline) and performs health checks.
- `Secrets`: Provides the source of truth for key availability.

Interfaces (Telegram, Bluesky, etc.) have no formal declaration mechanism at all. They are conditionally started based on environment variables in their respective supervisors. As the system grows—specifically with the introduction of TOML-based interface configurations—this ad-hoc, hardcoded validation will become a maintenance burden and a source of opaque startup failures.

## 2. Current State

The system currently employs a "validate-on-load" pattern for backends:
- **TOML Configs**: Declare `api_key_env`, `capabilities`, `performance`, and `structured_output`.
- **ConfigLoader**: Performs a "big-bang" validation during file loading, checking if secrets exist in `Myelin.Secrets`.
- **Secrets**: Resolves keys from `.secrets`, `.env`, or the system environment.
- **BackendPool**: Manages the lifecycle and status of these backends.
- **Interfaces**: Check for required environment variables (e.g., `TELEGRAM_BOT_TOKEN`) inside their `init/1` or via supervisor logic, leading to "silent" non-starts or crashes if requirements are missing.

## 3. Proposed Model: The Provider Manifest

Each provider (backend or interface) will declare a formal **Manifest** that separates its identity from its operational requirements.

### Manifest Components
- **Provides**: The set of capabilities, calling conventions, or platform access it offers (e.g., `[:routing, :tool_use]`, `:bluesky_access`).
- **Requires**: Hard dependencies that must be met for the provider to start.
    - `secrets`: A list of required secret keys (e.g., `["ANTHROPIC_API_KEY"]`).
    - `config`: Specific configuration values or paths.
    - `services`: External endpoints or local ports that must be reachable.
- **Constraints**: Minimum standards the environment or provider must meet.
    - `performance`: TTFT, tokens/sec, or context window minimums.
    - `features`: Grammar support (GBNF), JSON schema, or vision support.

### Resolution Lifecycle
At startup or configuration reload, a **Resolver** evaluates all manifests against the current `Registry` and `Secrets` state.
1. **Resolution**: Each requirement is checked.
2. **Status Assignment**:
    - `:operable`: All requirements and constraints are satisfied.
    - `:degraded`: Optional requirements are missing (e.g., a non-critical secret), but core functionality remains.
    - `:inoperable`: Hard requirements are missing; the provider will not be started.

### Pseudocode Example
```elixir
# Interface Manifest (future state)
def manifest do
  %{
    provides: [:bluesky_io],
    requires: %{
      secrets: ["BLUESKY_IDENTIFIER", "BLUESKY_PASSWORD"],
      services: ["localhost:8000"] # Bridge service
    },
    constraints: %{
      max_post_length: 300
    }
  }
end
```

## 4. When to Build

This system should be implemented **when interfaces migrate to TOML-based configurations**. 

Until there is a second major consumer of the declarative pattern, the current validation logic in `ConfigLoader` is sufficient and less complex. Building it now would be premature abstraction. The migration of interfaces to TOML creates the necessary pressure to unify how Myelin understands "can this component actually run?"

## 5. Design Principles

- **Predicates, Not Procedures**: Each requirement check must be a pure boolean check (e.g., `Secret.present?(key)`), not a side-effecting operation.
- **Fail Informatively**: If a provider is `:inoperable`, the system must log the specific missing requirement (e.g., `"Interface.Bluesky: missing secret 'BLUESKY_PASSWORD'"`).
- **Hot-Resolvable**: The resolver should re-evaluate manifests when `Secrets.reload()` is called, allowing the system to go from `:inoperable` to `:operable` without a full restart.
- **Zero Runtime Cost**: Resolution happens at startup or on explicit configuration change events. The hot path (event ingestion/routing) should only check the pre-calculated status.
