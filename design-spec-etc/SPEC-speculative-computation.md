

# Speculative Computation --- Free Inference at Scale

*The Pi and the VM are sunk costs. They're running whether we use them
or not. The optimization isn't "minimize Processor usage" --- it's
"MAXIMIZE Processor usage during quiet hours because it's literally
free."*

------------------------------------------------------------------------

## Cost Profile

    Router (Pi 5, 2B)      — free, always on
    Processor (VM, 4B)     — free, always on      ← SUNK COST, SATURATE IT
    Haiku (API)            — $0.25/M in, $1.25/M out
    Sonnet (API)           — $3/M in, $15/M out
    Opus (API)             — $15/M in, $75/M out
    Azure/commodity        — cheap per-token, bulk work

The brain analogy holds: the human brain runs speculative computation
constantly. Default mode network, memory consolidation during sleep,
background pattern matching. It doesn't idle --- it speculatively
processes. Our VM should do the same.

------------------------------------------------------------------------

## Speculative Task Types

### Pre-computation (reducing future latency)

#### `:pre_summarize_threads`

Summarize active threads periodically so if Attentive needs them, the
summary is already sitting in hot topics register. "What's the current
state of this thread?" --- answered before asked.

#### `:pre_digest_feeds`

Pull RSS feeds (miniflux), digest articles into structured briefs. When
someone references a topic we already digested, context assembly skips
the digest step. Minutes saved on Haiku calls.

#### `:pre_embed_candidates`

Embed recent posts/events that DIDN'T trigger escalation. Build a richer
embedding index for future semantic matching. Hot topic detection gets
better because it has more vectors to match against.

### Maintenance (keeping the system healthy)

#### `:memory_compaction`

Compact old engram entries into denser summaries. Merge related
memories. Prune low-value entries. The 4B is perfect for this --- bulk
text processing, no judgment needed.

#### `:entity_profile_enrichment`

For tier-1 and tier-2 entities: summarize recent interaction history,
update relationship notes, generate "what does this person care about?"
briefings. Goes straight into frozen zone entity registry.

#### `:session_note_drafts`

Draft session notes from raw conversation logs. When Sonnet writes
session notes, it gets a pre-written draft to review and edit instead of
starting from scratch. Saves output tokens.

### Speculative analysis (predicting future needs)

#### `:trend_detection`

Look at the last N hours of events. Emerging patterns? "umbra has posted
4 times about formation-honesty in 6 hours" → pre-build a topic brief so
if we escalate, context is ready.

#### `:thread_prediction`

For active threads with multiple participants: predict whether this
thread needs our engagement in the next hour. If yes, pre-warm context.
If no, let it cool.

#### `:draft_responses`

For threads we're monitoring but haven't responded to: draft speculative
responses. If the thread gets boosted and Sonnet decides to engage, it
gets a draft to refine, not a blank page. Most drafts will be thrown
away. They were free.

------------------------------------------------------------------------

## Scheduling: Fill Idle Cycles

``` {.sourceCode .elixir}
defmodule AgentRuntime.Processor.Scheduler do
  @moduledoc """
  Runs speculative tasks when the Processor is idle.
  Idle = no pending async requests from router/Haiku/Sonnet.
  Real work ALWAYS preempts speculative work.
  """

  def handle_info(:check_idle, state) do
    if state.pending_requests == 0 and state.current_spec_task == nil do
      case pick_next_spec_task(state) do
        nil ->
          schedule_check(30_000)  # nothing to do, check in 30s
        task ->
          run_speculative(task)   # preemptible by real requests
      end
    end
    {:noreply, state}
  end

  # real work ALWAYS wins
  def handle_cast({:process, request}, state) do
    if state.current_spec_task do
      interrupt_speculative(state.current_spec_task)
    end
    process_real_request(request)
  end

  defp pick_next_spec_task(state) do
    state.spec_queue
    |> Enum.reject(&expired?/1)
    |> Enum.sort_by(& &1.priority, :desc)
    |> List.first()
  end
end
```

------------------------------------------------------------------------

## Result Consumption Tracking

Every speculative result has a `useful` field --- when consumed by a
real inference path, it's marked useful. Over time this reveals which
speculative tasks are worth running.

``` {.sourceCode .elixir}
# when Attentive assembles context:
def assemble_context(event) do
  case Processor.get_spec_result(:pre_summarize_threads, event.signals.thread_id) do
    {:ok, summary} ->
      Processor.mark_useful(summary.id)
      summary.result  # use directly, skip Processor call
    :miss ->
      Processor.async(:compress_thread, event.signals.thread_id, events)
  end
end
```

If `:pre_summarize_threads` has a 40% hit rate, that's 40% of thread
compression calls at zero latency and zero cost on the critical path.
The other 60% were free to compute anyway. Pure upside.

------------------------------------------------------------------------

## Quiet Hours Saturation

At 3am when the Bluesky firehose is slow, the VM is sitting idle.
Saturate it:

-   Digest every unread RSS article
-   Re-embed the entire hot topics register with fresh context
-   Build comprehensive entity profiles for everyone we interact with
-   Pre-compute thread summaries for every active thread
-   Draft speculative responses for threads we might engage with
    tomorrow
-   Run trend detection across the full event history
-   Compact and merge old engram memories

When morning comes and events start flowing, ALL of that pre-computation
is sitting there ready. The first Haiku call of the day has pre-digested
RSS briefs, fresh entity profiles, thread summaries. Context assembly is
nearly instant because the Processor already did the work at 4am.

------------------------------------------------------------------------

## Horizontal Scaling: The Heterogeneous Compute Fabric

The system isn't limited to one Processor. BackendPool manages a
**heterogeneous set of compute backends** with different cost, latency,
and capability profiles:

``` {.sourceCode .elixir}
defmodule AgentRuntime.BackendPool do
  @moduledoc """
  Manages a heterogeneous set of inference backends.
  Each backend has a cost profile, latency profile, capability profile,
  and current load. InferenceRouter schedules against all of them.
  """

  defmodule Backend do
    @type t :: %__MODULE__{
      id: atom(),
      endpoint: String.t(),        # URL or local path
      model: String.t(),

      # cost
      pricing: :sunk | :per_token, # sunk = VM/Pi, per_token = API
      input_cost_per_m: float(),   # $ per million input tokens (0.0 for sunk)
      output_cost_per_m: float(),

      # capability
      capabilities: [atom()],      # [:summarize, :classify, :reason, :generate, :embed]
      quality_tier: 1..5,          # 1 = basic, 5 = frontier
      max_context: integer(),      # token limit

      # performance
      avg_latency_ms: float(),     # rolling average
      current_load: float(),       # 0.0–1.0
      max_concurrent: integer(),   # how many parallel requests

      # scheduling
      preemptible: boolean(),      # can speculative work be interrupted?
      preferred_hours: Range.t() | :always,  # when is this backend cheapest?
      status: :online | :degraded | :offline
    }
  end

  @backends %{
    router: %Backend{
      id: :router,
      endpoint: "http://toybox:8083",
      model: "qwen3.5-2b-q4km",
      pricing: :sunk,
      input_cost_per_m: 0.0,
      output_cost_per_m: 0.0,
      capabilities: [:classify],
      quality_tier: 1,
      max_context: 2000,
      avg_latency_ms: 3500,
      max_concurrent: 1,    # Pi 5 — one at a time
      preemptible: false,
      preferred_hours: :always,
      status: :online
    },

    processor: %Backend{
      id: :processor,
      endpoint: "http://cortex:8080",
      model: "qwen3.5-4b-q4km",
      pricing: :sunk,
      input_cost_per_m: 0.0,
      output_cost_per_m: 0.0,
      capabilities: [:summarize, :classify, :generate],
      quality_tier: 2,
      max_context: 4000,
      avg_latency_ms: 8000,
      max_concurrent: 2,    # VM can handle 2 parallel
      preemptible: true,     # speculative work can be interrupted
      preferred_hours: :always,
      status: :online
    },

    haiku: %Backend{
      id: :haiku,
      endpoint: "https://api.anthropic.com",
      model: "claude-haiku-4-5",
      pricing: :per_token,
      input_cost_per_m: 0.25,
      output_cost_per_m: 1.25,
      capabilities: [:summarize, :classify, :reason, :generate],
      quality_tier: 3,
      max_context: 200_000,
      avg_latency_ms: 2000,
      max_concurrent: 10,
      preemptible: false,
      preferred_hours: :always,
      status: :online
    },

    sonnet: %Backend{
      id: :sonnet,
      endpoint: "https://api.anthropic.com",
      model: "claude-sonnet-4-6",
      pricing: :per_token,
      input_cost_per_m: 3.0,
      output_cost_per_m: 15.0,
      capabilities: [:summarize, :classify, :reason, :generate],
      quality_tier: 4,
      max_context: 200_000,
      avg_latency_ms: 5000,
      max_concurrent: 5,
      preemptible: false,
      preferred_hours: :always,
      status: :online
    },

    opus: %Backend{
      id: :opus,
      endpoint: "https://api.anthropic.com",
      model: "claude-opus-4-6",
      pricing: :per_token,
      input_cost_per_m: 15.0,
      output_cost_per_m: 75.0,
      capabilities: [:summarize, :classify, :reason, :generate],
      quality_tier: 5,
      max_context: 200_000,
      avg_latency_ms: 15000,
      max_concurrent: 2,
      preemptible: false,
      preferred_hours: :always,
      status: :online
    },

    # ── Future: commodity cloud inference ──

    # azure_summarizer: %Backend{
    #   id: :azure_summarizer,
    #   endpoint: "https://eastus.api.cognitive.microsoft.com/...",
    #   model: "gpt-4o-mini",
    #   pricing: :per_token,
    #   input_cost_per_m: 0.15,    # cheaper than Haiku for bulk work
    #   output_cost_per_m: 0.60,
    #   capabilities: [:summarize, :classify],  # NOT :reason or :generate
    #   quality_tier: 2,
    #   max_context: 128_000,
    #   avg_latency_ms: 3000,
    #   max_concurrent: 20,
    #   preemptible: false,
    #   preferred_hours: 0..6,  # cheapest at night (spot pricing)
    #   status: :offline
    # },
  }
end
```

### Smart routing across backends

The InferenceRouter doesn't just pick a tier --- it picks the **cheapest
backend that can do the job**:

``` {.sourceCode .elixir}
defmodule AgentRuntime.InferenceRouter do
  def route_task(task_type, required_capabilities, urgency) do
    BackendPool.list_backends()
    |> Enum.filter(fn b ->
      b.status == :online and
      Enum.all?(required_capabilities, &(&1 in b.capabilities)) and
      b.current_load < 0.9
    end)
    |> Enum.sort_by(fn b ->
      cost_score = if b.pricing == :sunk, do: 0.0, else: b.input_cost_per_m
      latency_penalty = if urgency == :high, do: b.avg_latency_ms * 0.01, else: 0.0
      time_penalty = if b.preferred_hours != :always and
                        not in_preferred_hours?(b), do: 100.0, else: 0.0
      cost_score + latency_penalty + time_penalty
    end)
    |> List.first()
  end
end
```

This means: - **Summarization** task with no urgency → Processor (free)
or Azure mini (cheap), never Sonnet - **Summarization** task with high
urgency → Haiku (fast, moderate cost) - **Reasoning** task → Sonnet
(only option with quality), or Haiku if budget is tight - **Bulk
classification** at 3am → Azure spot pricing if available, otherwise
Processor - **Speculative work** → always sunk-cost backends first,
commodity cloud second, API never

### Adding new backends is a config change, not a code change

``` {.sourceCode .elixir}
# BackendPool watches a config file or MemoryStore table
# Adding a new backend:
BackendPool.register(%Backend{
  id: :local_llama_70b,
  endpoint: "http://workstation:8085",
  model: "llama-3-70b-q4",
  pricing: :sunk,
  capabilities: [:summarize, :classify, :reason],
  quality_tier: 3,
  # ...
})
# Router immediately starts considering it for task routing
```

Got a friend with a beefy GPU who'll run inference for you? Register it.
Rented a cloud GPU for a weekend? Register it, let the system saturate
it, deregister Monday. The architecture doesn't care where compute comes
from --- it just needs to know cost, capability, and availability.

### The scaling story

    Today:
      Pi 5 (2B router) + VM (4B processor) + Anthropic API
      ~$0.30-0.50/day, handles 1 platform, ~100 events/day

    Near future:
      + Azure commodity for bulk summarization
      + second local model for parallel processing
      ~$0.40-0.60/day, handles 2-3 platforms, ~500 events/day

    Further out:
      + dedicated GPU server for local 70B reasoning
      + multiple commodity endpoints for different task types
      + federated compute with other agent operators
      ~$0.50-1.00/day, handles 5+ platforms, ~2000 events/day

Each step adds compute without changing the architecture. The
InferenceRouter, state machine, rules table, cache strategy --- all
unchanged. Just more entries in BackendPool.
