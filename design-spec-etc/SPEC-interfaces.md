# Interfaces — Bidirectional Platform Adapters

*Ingesters only told half the story. The system needs to talk back.*

---

## The Problem

Stage 1 built ingesters: Bluesky, Syslog, Timer, Internal. Each one knows how to receive data from its platform and normalize it into `%Event{}`. But:

- Engaged broadcasts `{:engaged_response, event, text, rules}` via PubSub — nobody delivers it
- Interactive returns response text to the caller — but doesn't know it's Telegram, or a CLI, or a web UI
- There's no `%OutputEvent{}` — no formal schema for "the system wants to do something on a platform"
- Sonnet can't reason about output capabilities because they're not in its context

The fix: rename ingesters to **interfaces**. Each interface handles both directions for its platform. The runtime speaks `%Event{}` (inbound) and `%OutputEvent{}` (outbound). The interface translates.

---

## Interface Behaviour

```elixir
defmodule AgentRuntime.Interface do
  @moduledoc """
  Behaviour for bidirectional platform adapters.
  Each interface handles input (platform → runtime) and output (runtime → platform).
  """

  @doc """
  Returns the interface's capability manifest.
  This is a structured description of what the interface can do,
  suitable for injection into LLM context blocks.
  """
  @callback capabilities() :: %Capabilities{}

  @doc """
  Deliver an OutputEvent to the platform.
  Returns {:ok, delivery_receipt} or {:error, reason}.
  The receipt contains platform-specific confirmation (post URI, message ID, etc.)
  """
  @callback deliver(%OutputEvent{}) :: {:ok, map()} | {:error, term()}

  @doc """
  Check whether this interface can deliver a specific output action.
  Fast check against capability list — no network calls.
  """
  @callback can_deliver?(atom()) :: boolean()

  # Input side is NOT part of the behaviour — each interface handles
  # input differently (Port, GenServer, polling, webhooks). The only
  # contract is: call IngestionPipeline.ingest/1 with a valid %Event{}.
end
```

### Why input isn't in the behaviour

Input mechanisms are too heterogeneous to abstract:
- Bluesky: Port reading JSON lines from a TypeScript process
- Syslog: GenServer receiving cast messages
- Timer: self-scheduling GenServer
- Internal: module with direct function calls
- Future Telegram: webhook or polling via ex_gram
- Future SIEM: Kafka consumer, syslog-ng pipe, or API polling

Forcing these into one callback would be clunky. The input contract is simpler: produce `%Event{}` structs, feed them to `IngestionPipeline.ingest/1`. How you get the raw data is your problem.

The output side IS uniform enough to abstract: every interface that supports output receives an `%OutputEvent{}` and translates it into a platform API call.

---

## OutputEvent Schema

```elixir
defmodule AgentRuntime.OutputEvent do
  @type t :: %__MODULE__{
    id: String.t(),                    # ulid
    action: atom(),                    # :post | :reply | :react | :send_message | :alert | ...
    target_interface: atom(),          # :bluesky | :telegram | :siem | ...

    # addressing — where does this go?
    target: %{
      thread_id: String.t() | nil,     # reply to this thread
      recipient: String.t() | nil,     # DM/chat target
      channel: String.t() | nil,       # channel/room
      parent_event_id: String.t() | nil # reply to this specific event
    },

    # content — what are we saying/doing?
    content: %{
      text: String.t() | nil,          # primary text content
      format: atom(),                  # :plain | :markdown | :richtext
      media: [map()] | nil,            # attachments, images, etc.
      metadata: map()                  # platform-specific extras (facets, keyboard, etc.)
    },

    # provenance — why are we doing this?
    source_session_id: String.t() | nil,
    source_event_id: String.t() | nil, # the inbound event that triggered this
    decided_by: atom(),                # :sonnet | :haiku | :interactive
    confidence: float() | nil,         # model's confidence in this output

    # scheduling
    priority: :low | :normal | :urgent,  # queue ordering + delivery urgency

    # lifecycle
    status: :pending | :delivering | :delivered | :failed | :rejected,
    created_at: DateTime.t(),
    delivered_at: DateTime.t() | nil,
    delivery_receipt: map() | nil,     # platform-specific confirmation
    retry_count: non_neg_integer()
  }
end
```

---

## Capability Manifests

This is the key innovation. Each interface declares what it can do, and these declarations go into the agent's context so it knows what actions are available.

```elixir
defmodule AgentRuntime.Interface.Capabilities do
  @type t :: %__MODULE__{
    interface: atom(),                  # :bluesky | :telegram | :siem | ...
    direction: :bidirectional | :input_only | :output_only,

    # what actions can this interface perform?
    actions: [%{
      action: atom(),                   # :post | :reply | :react | :send_message | ...
      description: String.t(),          # human-readable, for LLM context
      constraints: %{
        max_length: non_neg_integer() | nil,
        formats: [atom()],             # [:plain, :markdown, :richtext]
        rate_limit: String.t() | nil,  # "30/hour", "300/day", etc.
        requires_target: boolean(),    # needs a thread_id or recipient?
        supports_media: boolean(),
        supports_threading: boolean(),
        requires_approval: boolean()   # should go through approval queue?
      }
    }],

    # what event kinds does this interface produce?
    input_kinds: [atom()],             # [:mention, :reply, :like, :follow, ...]

    # platform identity
    platform_name: String.t(),         # "Bluesky", "Telegram", "Syslog"
    platform_identity: String.t() | nil, # our handle/username on this platform
    audience: String.t(),              # "public" | "private" | "internal"
  }
end
```

### Context block rendering

The capability manifest renders into a context block for the frozen zone:

```elixir
defmodule AgentRuntime.Interface.ContextRenderer do
  @doc """
  Renders all interface capabilities into a text block
  suitable for Layer 1 (situational context, 1-hour cache).
  """
  def render_capabilities(interfaces) do
    interfaces
    |> Enum.map(&render_one/1)
    |> Enum.join("\n\n")
  end

  defp render_one(%Capabilities{} = cap) do
    actions_text = cap.actions
      |> Enum.map(fn a ->
        constraints = format_constraints(a.constraints)
        "  - #{a.action}: #{a.description}#{constraints}"
      end)
      |> Enum.join("\n")

    """
    INTERFACE: #{cap.platform_name} (#{cap.interface})
    Identity: #{cap.platform_identity || "n/a"}
    Audience: #{cap.audience}
    Direction: #{cap.direction}
    Actions:
    #{actions_text}
    """
  end

  defp format_constraints(c) do
    parts = []
    parts = if c.max_length, do: parts ++ ["max #{c.max_length} chars"], else: parts
    parts = if c.rate_limit, do: parts ++ ["rate: #{c.rate_limit}"], else: parts
    parts = if c.requires_approval, do: parts ++ ["requires approval"], else: parts
    if parts == [], do: "", else: " [#{Enum.join(parts, ", ")}]"
  end
end
```

### Example rendered context block

This is what Sonnet sees in Layer 1:

```
AVAILABLE INTERFACES:

INTERFACE: Bluesky (@myelin.bsky.social)
Identity: @myelin.bsky.social
Audience: public
Direction: bidirectional
Actions:
  - post: Create a new top-level post [max 300 chars, rate: 30/hour]
  - reply: Reply to a thread [max 300 chars, requires thread target]
  - like: Like a post
  - repost: Repost/boost a post [rate: 30/hour]
  - follow: Follow a user

INTERFACE: Telegram (MyelinBot)
Identity: @MyelinBot
Audience: private
Direction: bidirectional
Actions:
  - send_message: Send a message to a chat [max 4096 chars, markdown]
  - edit_message: Edit a previously sent message
  - send_photo: Send an image with caption
  - reply_keyboard: Present options as inline keyboard buttons
  - pin_message: Pin a message in a chat

INTERFACE: Syslog
Identity: n/a
Audience: internal
Direction: input_only
Actions:
  (none — input only)

INTERFACE: Internal
Identity: system
Audience: internal
Direction: bidirectional
Actions:
  - emit_event: Emit an internal system event
  - schedule_task: Queue a deferred task
  - adjust_state: Request state machine transition
```

The model now knows: "I can post to Bluesky (300 chars, public) or send a Telegram message (4096 chars, private, markdown). Syslog is read-only. If I want to share something long, Telegram. If I want to engage publicly, Bluesky."

---

## Concrete Interface Specs

### Bluesky Interface

```elixir
defmodule AgentRuntime.Interface.Bluesky do
  @behaviour AgentRuntime.Interface

  # Input: Port wrapper reading JSON lines (existing code, mostly unchanged)
  # Output: ATProto API calls via the TypeScript process or direct HTTP

  @impl true
  def capabilities do
    %Capabilities{
      interface: :bluesky,
      direction: :bidirectional,
      actions: [
        %{action: :post, description: "Create a new top-level post",
          constraints: %{max_length: 300, formats: [:plain], rate_limit: "30/hour",
            requires_target: false, supports_media: true, supports_threading: false,
            requires_approval: false}},
        %{action: :reply, description: "Reply in a thread",
          constraints: %{max_length: 300, formats: [:plain], rate_limit: "30/hour",
            requires_target: true, supports_media: true, supports_threading: true,
            requires_approval: false}},
        %{action: :like, description: "Like a post",
          constraints: %{max_length: nil, formats: [], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: false}},
        %{action: :repost, description: "Repost/boost a post",
          constraints: %{max_length: nil, formats: [], rate_limit: "30/hour",
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: false}},
        %{action: :follow, description: "Follow a user",
          constraints: %{max_length: nil, formats: [], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: true}}  # follow is consequential, approval recommended
      ],
      input_kinds: [:mention, :reply, :like, :repost, :follow, :message],
      platform_name: "Bluesky",
      platform_identity: "@myelin.bsky.social",
      audience: "public"
    }
  end

  @impl true
  def deliver(%OutputEvent{action: :post} = output) do
    # Validate constraints
    # Call ATProto createRecord
    # Return {:ok, %{uri: ..., cid: ...}}
  end

  @impl true
  def deliver(%OutputEvent{action: :reply} = output) do
    # Resolve parent URI from target.parent_event_id
    # Call ATProto createRecord with reply ref
    # Return {:ok, %{uri: ..., cid: ...}}
  end

  # ... etc

  @impl true
  def can_deliver?(action) do
    action in [:post, :reply, :like, :repost, :follow]
  end
end
```

### Telegram Interface

```elixir
defmodule AgentRuntime.Interface.Telegram do
  @behaviour AgentRuntime.Interface

  # Input: ex_gram webhook/polling → events
  # Output: ExGram.send_message/2 etc.

  @impl true
  def capabilities do
    %Capabilities{
      interface: :telegram,
      direction: :bidirectional,
      actions: [
        %{action: :send_message, description: "Send a text message to a chat",
          constraints: %{max_length: 4096, formats: [:plain, :markdown], rate_limit: "30/second",
            requires_target: true, supports_media: false, supports_threading: true,
            requires_approval: false}},
        %{action: :edit_message, description: "Edit a previously sent message",
          constraints: %{max_length: 4096, formats: [:plain, :markdown], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: false}},
        %{action: :send_photo, description: "Send an image with optional caption",
          constraints: %{max_length: 1024, formats: [:plain], rate_limit: nil,
            requires_target: true, supports_media: true, supports_threading: false,
            requires_approval: false}},
        %{action: :reply_keyboard, description: "Present inline keyboard buttons for user choice",
          constraints: %{max_length: 4096, formats: [:plain, :markdown], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: false}},
        %{action: :pin_message, description: "Pin a message in a chat",
          constraints: %{max_length: nil, formats: [], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: false}}
      ],
      input_kinds: [:message],
      platform_name: "Telegram",
      platform_identity: "@MyelinBot",
      audience: "private"
    }
  end

  @impl true
  def can_deliver?(action) do
    action in [:send_message, :edit_message, :send_photo, :reply_keyboard, :pin_message]
  end
end
```

### Syslog Interface (input-only)

```elixir
defmodule AgentRuntime.Interface.Syslog do
  @behaviour AgentRuntime.Interface

  @impl true
  def capabilities do
    %Capabilities{
      interface: :syslog,
      direction: :input_only,
      actions: [],
      input_kinds: [:log_event, :sigma_hit],
      platform_name: "Syslog",
      platform_identity: nil,
      audience: "internal"
    }
  end

  @impl true
  def deliver(_), do: {:error, :input_only}

  @impl true
  def can_deliver?(_), do: false
end
```

### Future: SIEM Interface

```elixir
# Example of a security-focused interface with mixed capabilities
defmodule AgentRuntime.Interface.SIEM do
  @behaviour AgentRuntime.Interface

  @impl true
  def capabilities do
    %Capabilities{
      interface: :siem,
      direction: :bidirectional,
      actions: [
        %{action: :create_ticket, description: "Create an incident ticket in the SIEM",
          constraints: %{max_length: 10_000, formats: [:markdown], rate_limit: "10/hour",
            requires_target: false, supports_media: false, supports_threading: false,
            requires_approval: true}},
        %{action: :update_ticket, description: "Add notes to an existing incident ticket",
          constraints: %{max_length: 5000, formats: [:markdown], rate_limit: nil,
            requires_target: true, supports_media: false, supports_threading: true,
            requires_approval: false}},
        %{action: :isolate_host, description: "Request network isolation of a host",
          constraints: %{max_length: nil, formats: [], rate_limit: "5/day",
            requires_target: true, supports_media: false, supports_threading: false,
            requires_approval: true}}  # ALWAYS requires approval — destructive action
      ],
      input_kinds: [:sigma_hit, :log_event, :alert],
      platform_name: "SIEM",
      platform_identity: "myelin-agent",
      audience: "internal"
    }
  end
end
```

---

## Output Pipeline

```
Sonnet/Haiku/Interactive decides to output
         │
         ▼
  OutputEvent created
  (action, target, content, provenance)
         │
         ▼
  ┌──────────────────┐
  │ OutputPipeline    │ ← GenServer, analogous to IngestionPipeline
  │                   │
  │ 1. Validate       │ ← does target interface support this action?
  │ 2. Constrain      │ ← enforce max_length, format rules
  │ 3. Prioritize     │ ← :urgent jumps queue, :low defers to idle
  │ 4. Approve?       │ ← if requires_approval: queue for human review
  │ 5. Rate limit     │ ← check against interface rate limits
  │ 6. Deliver        │ ← call interface.deliver/1
  │ 7. Receipt        │ ← store delivery confirmation
  │ 8. Feedback event │ ← emit internal event for delivery result
  └──────────────────┘
         │
         ▼
  Internal event: {:output_delivered, output_event_id, receipt}
  or:             {:output_failed, output_event_id, reason}
```

### Approval queue

Some actions are consequential enough to need human approval before delivery. The `requires_approval` flag on each action capability controls this.

Default approval requirements:
- **Always approve:** `:isolate_host`, `:follow` (on public platforms)
- **Configurable:** `:post` (public speech), `:create_ticket` (creates work for others)
- **Never approve:** `:reply` (in a thread we're already in), `:like`, `:send_message` (private to operator)

When approval is required:
1. OutputEvent enters `:pending_approval` status
2. Operator notified via Telegram (or whatever primary interface is configured)
3. Operator approves/rejects
4. If approved, OutputEvent enters normal delivery pipeline
5. If rejected, OutputEvent marked `:rejected`, reason logged

---

## Interface Registry

All active interfaces are registered and queryable:

```elixir
defmodule AgentRuntime.Interface.Registry do
  @moduledoc """
  Registry of active interfaces and their capabilities.
  Used by the context assembler to build capability blocks,
  and by the output pipeline to route deliveries.
  """

  def list_interfaces() :: [%Capabilities{}]
  def get_interface(name :: atom()) :: %Capabilities{} | nil
  def can_deliver?(interface :: atom(), action :: atom()) :: boolean()
  def find_interface_for_action(action :: atom()) :: [atom()]

  # Renders all capabilities for injection into Layer 1
  def render_context_block() :: String.t()
end
```

---

## Integration with Existing Architecture

### Cache Layer 1 (Situational)

Interface capabilities go into Layer 1 — they change rarely (only when interfaces are added/removed or reconfigured), so they benefit from the 1-hour cache:

```
LAYER 1: SITUATIONAL (1-hour cache)
├── Rules snapshot
├── Entity registry (top-20)
├── Cost summary
└── Interface capabilities    ← NEW
```

Token budget for capabilities: ~300-500 tokens (the rendered block above is ~250 tokens for 4 interfaces). Fits easily within Layer 1's 1600 token budget.

### Engaged layer

Sonnet currently has rule-writing tools. Add output tools:

```elixir
# New tool for Engaged sessions
%{
  name: "output",
  description: "Send output through an interface",
  input_schema: %{
    type: "object",
    properties: %{
      interface: %{type: "string", enum: ["bluesky", "telegram"]},
      action: %{type: "string"},
      text: %{type: "string"},
      target_thread_id: %{type: "string"},
      target_recipient: %{type: "string"}
    },
    required: ["interface", "action", "text"]
  }
}
```

Sonnet sees the capability block, knows what's available, and uses the output tool with valid parameters. The tool handler creates an `%OutputEvent{}` and feeds it to the output pipeline.

### Interactive layer

Interactive already returns response text. The Telegram bridge (Phase 8B) becomes the Telegram interface's output delivery path. When Interactive produces a response, it creates an OutputEvent targeted at the interface that originated the conversation.

### Delivery receipts as internal events

When output is delivered, the receipt feeds back as an internal event:
- Bluesky post delivered → `Internal.emit(:output_delivered, %{interface: :bluesky, uri: "at://..."})`
- This can trigger rule writes: Sonnet might `boost_thread` on the thread it just posted in
- Session records the output event ID for audit

---

## Migration Path

### What changes

| Current | New |
|---|---|
| `AgentRuntime.Ingester.Bluesky` | `AgentRuntime.Interface.Bluesky` |
| `AgentRuntime.Ingester.Syslog` | `AgentRuntime.Interface.Syslog` |
| `AgentRuntime.Ingester.Timer` | `AgentRuntime.Interface.Timer` (input-only, keeps current behavior) |
| `AgentRuntime.Ingester.Internal` | `AgentRuntime.Interface.Internal` |
| `AgentRuntime.Ingester.EventEnricher` | `AgentRuntime.Interface.EventEnricher` (unchanged, shared) |
| (nothing) | `AgentRuntime.Interface` behaviour |
| (nothing) | `AgentRuntime.OutputEvent` |
| (nothing) | `AgentRuntime.Interface.Capabilities` |
| (nothing) | `AgentRuntime.Interface.Registry` |
| (nothing) | `AgentRuntime.OutputPipeline` |
| `Ingester.Supervisor` | `Interface.Supervisor` |
| `IngestionPipeline` | `EventPipeline` (or keep name, it still does ingestion) |

### What doesn't change

- `%Event{}` schema — untouched
- `EventEnricher` — untouched (just moved to new namespace)
- `InferenceRouter` — untouched (still receives enriched events)
- `StateMachine` — untouched
- Cache layers — Layer 1 gets a new section, no structural change
- The input pathway is identical — interfaces still produce `%Event{}` and feed them to the pipeline

### Implementation order

1. Define `%OutputEvent{}` struct and `%Capabilities{}` struct
2. Define `Interface` behaviour
3. Add `capabilities/0` to existing ingesters (rename later)
4. Build `OutputPipeline` GenServer
5. Add output tool to Engaged
6. Wire Telegram interface (Phase 8) as first real bidirectional interface
7. Rename `Ingester.*` → `Interface.*` (mechanical, do last)

---

## Design Properties

- **Capabilities are data, not code.** The model sees a structured text block describing what it can do. It doesn't call platform APIs directly — it creates OutputEvents that the pipeline validates and delivers.
- **Input is heterogeneous, output is uniform.** Input mechanisms vary wildly (Ports, casts, polling, webhooks). Output is always: create OutputEvent → validate → deliver. This asymmetry is intentional.
- **Approval is per-action, not per-interface.** A Bluesky interface might allow auto-reply but require approval for new posts. The granularity is at the action level.
- **Delivery receipts close the loop.** Output delivery feeds back as internal events, creating a full cycle: input → processing → output → confirmation → potential rule adjustment.
- **Context blocks are cheap.** The entire capability manifest for 4 interfaces is ~300 tokens. It fits in Layer 1's 1-hour cache and costs effectively nothing after the first write.
