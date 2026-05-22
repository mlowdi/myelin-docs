# Manpage Contributor Guide

How to add, format, and organize documentation that Myelin's agent reads at runtime.

## How It Works

`Myelin.ReadDocs` loads all `.md` files from `priv/manpages/` at startup. The file path becomes the lookup key:

```
priv/manpages/system/routing.md  →  agent calls read_docs("system/routing")
priv/manpages/tools/web_fetch.md →  agent calls read_docs("tools/web_fetch")
```

No registration, no config, no code changes. Drop a `.md` file in the right directory and call `ReadDocs.reload()` (or restart). The topic tree in the agent's frozen context updates automatically.

## Adding a New Manpage

1. Create a `.md` file in the appropriate `priv/manpages/` subdirectory
2. Follow the format rules below
3. Run `ReadDocs.reload()` from IEx (or restart the system)
4. Verify with `ReadDocs.lookup("your/path")` and check `ReadDocs.topic_tree()`

That's it. No other files to touch.

## Directory Structure

```
priv/manpages/
  system/          # How Myelin works internally (reference)
  manage/          # Things the agent can actively manage
    budget/        # Budget inspection and conservation
    entities/      # Entity registry operations
    rules/         # Rules table operations
  tools/           # Async operations the agent can trigger
  context/         # How the agent's prompt is assembled
  troubleshooting/ # What to do when things go wrong
```

**Choosing the right category:**

- Is it a concept the agent needs to understand? → `system/`
- Is it something the agent can invoke? → `tools/`
- Is it something the agent can inspect or modify? → `manage/`
- Is it about how the agent's own context works? → `context/`
- Is it about diagnosing problems? → `troubleshooting/`

Nest one level deep within categories: `manage/budget/summary.md`, not `manage/budget/reports/daily/summary.md`. Two levels max below `priv/manpages/`.

## Format Rules

### System / Reference Pages

```markdown
# category/name

One-sentence description of what this page covers.

## Section Title

Concise explanation. Write for an LLM that needs to reason about the
system — it should understand *what* happens and *why*, not implementation
details.

## Another Section

- Bullet points for lists of things
- Use concrete values where they exist (thresholds, multipliers, limits)
- Reference other manpages by path: "See system/routing for details"
```

### Tool Pages

```markdown
# tools/name

One-sentence description of what the tool does.

## Calling Convention

Emit via structured output:
```json
{"tool": "tool_name", "args": {"param": "value"}}
```

## Parameters

- `param_name` — Description. Type, constraints if relevant.
- `optional_param` — Description. Default value if applicable.

## Returns

A JSON object containing:
- `field` — What this field means.

## Notes

- When to use this tool.
- What it costs (if relevant to budget).
- Where the results are stored.
- Constraints or limitations.
```

### Manage Pages

```markdown
# manage/category/action

One-sentence description.

## Calling Convention

(same as tools — JSON structured output)

## Parameters

(same as tools)

## Effects

- What changes in the system when this is called.
- Side effects (PubSub broadcasts, state changes).

## Notes

- When the agent should use this.
- Guardrails or restrictions.
```

### Troubleshooting Pages

```markdown
# troubleshooting/problem_name

One-sentence description of the problem.

## Symptoms

How the agent would notice this problem.

## Diagnosis

What to check and how to determine the cause.

## Actions

What the agent should do:
- Immediate actions (notify operator, enter conservation mode)
- Information to include in the notification
- What NOT to do

## Recovery

How the system recovers (automatic probe, manual intervention needed).
```

## Style Rules

1. **30-80 lines per page.** If it's longer, split it. If it's shorter, it probably doesn't need its own page.
2. **No emojis.** The agent reads these as working reference, not decoration.
3. **No preamble.** First line is `# path`, second line is the description. No "Welcome to..." or "This document explains..."
4. **Concrete over abstract.** Use actual values: "salience threshold 0.7" not "a high salience threshold." Use actual module names: "stored in MemoryStore" not "stored in the persistence layer."
5. **Reference other pages by path.** "See system/routing" tells the agent it can `read_docs("system/routing")` for more.
6. **Write for an LLM.** The reader is an AI agent deciding what to do. It needs: what is this, when do I use it, what happens when I do, what could go wrong. It does NOT need: implementation history, design rationale, alternative approaches considered.
7. **No code snippets** (except calling conventions). The agent doesn't read Elixir. It reads JSON tool calls and natural language descriptions.

## When to Write a Manpage

If you're building a feature that:

- **Adds a tool the agent can call** → write a `tools/` page
- **Adds a manage operation** → write a `manage/` page
- **Changes system behavior the agent should know about** → update or add a `system/` page
- **Introduces a new failure mode** → write a `troubleshooting/` page
- **Adds a new capability** → write a `system/capabilities` update AND any tool pages for what the capability enables

The rule: **if the agent needs to know about it to do its job, it needs a manpage.** If it's purely internal implementation, it doesn't.

## Future: Capability-Gated Manpages

Currently all manpages load unconditionally. Future work will gate manpages on the capability graph — if no `:vision` backend is connected, the vision tool manpage won't load, and the agent won't know to try. This prevents hallucinated tool calls for capabilities that don't exist.

When this lands, manpages that depend on a capability will declare it in frontmatter:

```markdown
---
requires: [vision, feature_extraction]
---
# tools/describe_image

Describe the contents of an image using a vision-capable model.
...
```

Until then, all manpages load. Keep this future gating in mind when writing pages for capability-dependent features.
