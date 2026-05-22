

# Event Schema --- `%Event{}`

*The blood type of the whole system. Every process speaks Event.*

------------------------------------------------------------------------

## Design Constraints

-   EventIngesters normalize platform-specific data into this one shape
-   Everything downstream only speaks Event
-   Carries enough for routing decisions WITHOUT carrying full payload
-   Signals are pre-computed at ingestion time --- router reads
    categorical inputs, never does lookups

------------------------------------------------------------------------

## Schema

``` {.sourceCode .elixir}
defmodule AgentRuntime.Event do
  @type t :: %__MODULE__{
    # identity
    id: String.t(),                    # ulid — sortable, unique
    source: atom(),                    # :bluesky | :signal | :syslog | :cron | :internal
    source_id: String.t() | nil,      # platform-native ID (rkey, message ID, etc.)

    # classification (set by ingester, refined by router)
    kind: atom(),                      # :message | :mention | :reply | :follow | :like
                                       # | :log_event | :sigma_hit | :timer | :cache_available

    # who
    sender: String.t() | nil,          # normalized handle/identifier
    sender_tier: 1..3 | nil,           # looked up from entity registry at ingestion time

    # content (lightweight — full payload stays in ingester or is fetched on demand)
    summary: String.t(),               # ≤280 chars, human-readable, router-consumable
    token_estimate: non_neg_integer(), # cheap estimate of full payload size
    payload_ref: reference() | nil,    # opaque ref to fetch full content if needed

    # context signals (pre-computed by ingester, cheap for router to read)
    signals: %{
      thread_id: String.t() | nil,       # conversation threading
      parent_id: String.t() | nil,       # immediate parent event
      hot_topic_match: float() | nil,    # 0.0–1.0, checked against hot topics register
      entity_salience: float() | nil,    # from entity registry
      keyword_hits: [atom()],            # matched against active rules table keywords
    },

    # metadata
    timestamp: DateTime.t(),           # normalized to UTC always
    ingested_at: DateTime.t(),         # when we received it
    ttl: non_neg_integer() | :infinity # seconds until this event is irrelevant
  }
end
```

------------------------------------------------------------------------

## Design Decisions

### `summary` instead of full content

The router is a 2B model with a \~100 token budget for terse routing. It
gets the summary, the signals, the sender tier. That's it. If it
escalates, the Attentive layer fetches full content via `payload_ref`.
This is the cache hierarchy in action --- L1 gets the summary, L2 gets
the full thing.

### `sender_tier` at ingestion

The entity registry lookup happens ONCE when the event enters the
system, not every time something inspects it. Tier-1 sender → immediate
salience boost. This is the "phone buzz" vs "thought" distinction from
Interactive mode.

### `signals` as pre-computed map

The router should NOT be doing semantic search or registry lookups. The
ingester and MemoryStore do that work at event creation time and stamp
the results onto the event. Router reads categorical inputs, never raw
numbers. Straight from the architecture doc.

### `payload_ref` is opaque

Could be a process mailbox, an ETS key, a function closure, an atproto
URI. The point is that fetching full content is a *decision*, not a
default. Most events in Dormant/Monitoring state will never have their
payload fetched.

### `kind` as atom vocabulary

Extensible but not stringly-typed. Each EventIngester maps platform
concepts to this shared vocabulary. A Bluesky reply and a Signal message
can both be `:message`. A Sigma alert and a Bluesky mention have
completely different kinds. The router's rules operate on kinds.

------------------------------------------------------------------------

## Ingestion Cost (Bluesky example)

The enrichment pipeline for a single event is extremely cheap:

1.  **Parse** --- atproto gives structured data, basically free
2.  **Entity lookup** --- sqlite query on handle, get tier + salience.
    Microseconds
3.  **Keyword match** --- rules table keywords against post text, string
    matching. Free
4.  **Hot topic check** --- embed summary (one HTTP call to
    snowflake-arctic on LM Studio, ≤2k chars), cosine similarity against
    in-memory hot topics. sqlite-vec is fast
5.  **Stamp signals, emit Event** --- struct creation

Steps 2--4 run in parallel (`Task.async`). Total ingestion time
estimated \<100ms on toybox (nvme + local embedding server + sqlite).

The vast majority of events in Dormant/Monitoring just stop here. Router
glances at signals, drops it. Never touches an API. Never fetches full
payload.

------------------------------------------------------------------------

## Example: Bluesky Ingester

``` {.sourceCode .elixir}
defmodule AgentRuntime.Ingester.Bluesky do
  @moduledoc """
  Connects to Bluesky firehose (or polling endpoint).
  Normalizes atproto events into %Event{}.
  Does ALL cheap enrichment at ingestion time.
  """

  defp enrich(raw_post) do
    summary = truncate(raw_post.text, 280)
    lang = List.first(raw_post.langs) || "en"
    cleaned = strip_stopwords(summary, lang)

    # parallel — these are independent lookups
    entity_task = Task.async(fn -> MemoryStore.lookup_entity(raw_post.author.handle) end)
    embed_task = Task.async(fn -> embed_and_match_topics(cleaned) end)
    keyword_task = Task.async(fn -> MemoryStore.match_keywords(raw_post.text) end)

    entity = Task.await(entity_task)
    {hot_topic_score, _embedding} = Task.await(embed_task)
    keyword_hits = Task.await(keyword_task)

    %Event{
      id: ULID.generate(),
      source: :bluesky,
      source_id: raw_post.uri,
      kind: classify_kind(raw_post),
      sender: raw_post.author.handle,
      sender_tier: entity && entity.tier,
      summary: summary,
      token_estimate: div(String.length(raw_post.text), 4),
      payload_ref: {:bluesky, raw_post.uri},
      signals: %{
        thread_id: raw_post.reply && raw_post.reply.root.uri,
        parent_id: raw_post.reply && raw_post.reply.parent.uri,
        hot_topic_match: hot_topic_score,
        entity_salience: entity && entity.salience,
        keyword_hits: keyword_hits
      },
      timestamp: raw_post.created_at |> DateTime.from_iso8601!(),
      ingested_at: DateTime.utc_now(),
      ttl: ttl_for_kind(classify_kind(raw_post))
    }
  end
end
```

### Ingester ↔︎ Runtime seam

The Bluesky ingester is a TypeScript service (atproto SDK is TS-native).
Communication with the Elixir runtime via:

-   **v1: Elixir Port** --- spawns TS process, reads JSON lines over
    stdio. Zero infrastructure. Elixir's port system is designed for
    exactly this.
-   **v2: Unix socket** --- if ingester needs to restart independently
-   **v3: Redis streams / AMQP** --- if we need multiple consumers or
    replay. Probably never.

Start with Port. Promote when it hurts.
