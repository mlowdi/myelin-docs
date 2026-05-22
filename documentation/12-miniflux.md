# Miniflux Interface

The `Myelin.Interface.Miniflux` module provides a bidirectional interface to the Miniflux RSS aggregator.

## Overview

Miniflux integration supports:
1. **Ingestion**: Polling for unread articles and injecting them into the `EventPipeline`.
2. **Action**: Starring (bookmarking) specific articles for the operator to review later.

This interface uses the `Req` library for HTTP communication and does not require an external Port or bridge.

## Polling Flow

The interface polls the Miniflux API at a configurable interval (default: 60s).

1. `GET /v1/entries?status=unread&limit=50` retrieves recent unread articles.
2. For each retrieved article, a `Myelin.Event` is created and ingested.
3. `PUT /v1/entries` is called with `status: "read"` to mark the polled articles as read in Miniflux, preventing duplicate ingestion.

Exponential backoff is implemented for handling failures (HTTP errors or connection issues).

## Starring Flow

The interface supports the `:star_entry` output action.

1. An `OutputEvent` with `action: :star_entry` and `target: %{entry_id: id}` is sent to the interface.
2. The interface calls `PUT /v1/entries/{entry_id}/bookmark` to toggle the bookmark status.
3. In Miniflux, this marks the entry as "Starred", surfacing it for the operator.

## Configuration

The interface is managed by `Myelin.Interface.Supervisor` and starts automatically if `MINIFLUX_API_KEY` is present.

### Secrets
- `MINIFLUX_API_KEY`: Required. Your Miniflux API token.
- `MINIFLUX_BASE_URL`: Optional. Defaults to `http://toybox:8082`.

### Application Config
```elixir
config :myelin, Myelin.Interface.Miniflux,
  base_url: "http://your-miniflux-instance",
  poll_interval: 60_000
```

## Supervisor Wiring

The interface is wired via `maybe_add_miniflux/1` in `Myelin.Interface.Supervisor`. It checks for the existence of the API key before adding the module to the supervision tree.
