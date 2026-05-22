# Interactive Memory Tools (Layer 3)

This document covers the architecture and implementation details of the ephemeral memory tools feature (#124) added to the `Myelin.Interactive` GenServer.

## Purpose and Scope

Interactive sessions often require the agent to keep track of multi-step tasks or collect notes across several conversational turns. Before this feature, the agent had to rely entirely on the rolling conversation window (hot zone) or commit permanent records to the `MemoryStore`.

The Interactive memory tools provide a middle ground: **session-scoped, ephemeral state**. 
- It lives entirely within the `Myelin.Interactive` GenServer state (`state.todos` and `state.scratchpad`).
- It is injected dynamically into the context window as "Layer 3".
- It is strictly ephemeral — it is cleared completely when a session is closed and re-opened. It is never persisted to SQLite.

## The Tool Set

The Interactive GenServer exposes five tools to the inference engine during the `interactive_turn` and `interactive_interrupt` loops:

1.  **`add_todo`**: Appends a string to the `state.todos` list.
    - Input: `%{text: "..."}`
2.  **`remove_todo`**: Removes a todo item by its 1-based index.
    - Input: `%{index: 1}`
3.  **`clear_todos`**: Empties the `state.todos` list.
    - Input: `%{}`
4.  **`set_scratchpad`**: Overwrites the entire `state.scratchpad` string.
    - Input: `%{text: "..."}`
5.  **`append_scratchpad`**: Appends a string to the existing `state.scratchpad`, separated by a newline.
    - Input: `%{text: "..."}`

## Layer 3 Context Injection

The state of the todos and scratchpad is surfaced to the agent via **Layer 3** of the context assembly process.

In `Myelin.Interactive.build_layers/1`, the system calls `render_layer3(todos, scratchpad)`.
- If both the todo list and scratchpad are empty, `render_layer3/2` returns an empty string, and `layer3` is set to `nil`.
- If either has content, it formats them into clear Markdown blocks (e.g., `My todos:\n1. ...` and `Scratchpad:\n...`).

`Myelin.Cache.Assembler.build_messages/4` was extended for `:interactive` and `:engaged` tiers to accept this optional fourth layer. If `layer3` is present, it is appended as an additional `system` message *before* the hot zone. Because it is highly dynamic, Layer 3 is not wrapped in an `ephemeral` cache control block (unlike Layers 0 and 1).

## Execution Loop (`run_loop/4`)

The tool execution is handled by `Myelin.Interactive.run_loop/4`, which orchestrates a multi-turn conversation with the inference engine (up to a maximum of 5 turns per user message).

1.  The Inference engine returns a `Response` with `stop_reason == :tool_use`.
2.  `execute_tools/2` pattern matches on the tool names, updates the GenServer state (`acc_state`), and generates a list of `tool_result` blocks.
3.  The assistant's tool call and the system's `tool_result` (acting as the user) are appended to the `messages` array.
4.  `run_loop/4` recurses with `remaining_turns - 1` and the updated `new_state`.

This loop ensures that the agent immediately sees the result of its tool calls (e.g., "Todo added: ...") in the same interaction cycle, and the updated Layer 3 will be present in the next full user interaction.

## Testing Approach

Because these tools interact directly with the `Myelin.Interactive` GenServer state, they are primarily tested in `test/myelin/interactive_memory_tools_test.exs` and `test/myelin/interactive_test.exs`.

Tests should:
- Mock the inference module to simulate returning `stop_reason == :tool_use` with the specific tool payload.
- Verify that `execute_tools/2` updates the state correctly.
- Verify that `render_layer3/2` correctly formats the output or returns `""` when empty.
- Ensure that `close_session` properly resets `todos` to `[]` and `scratchpad` to `""`.
