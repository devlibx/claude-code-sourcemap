# Architectural Masterclass: The Code Editing & Agentic Loop

This manual provides a high-resolution technical analysis of the recursive orchestration engine behind Claude Code. It explains how the system manages state, concurrency, and safety during autonomous code modifications.

---

## 1. The Orchestration Engine: `src/query.ts`
At the heart of Claude Code is a **Recursive Generator Function**. Unlike a traditional request-response model, the `query` function manages a "trajectory" of multiple turns.

### Trajectory Accumulation
Every time Claude uses a tool, the loop in `query.ts` performs the following:
1. **Tool Dispatch:** Parses the `AssistantMessage` for `tool_use` blocks.
2. **Result Collection:** Executes the tools and wraps their output in a `UserMessage` with `type: 'tool_result'`.
3. **History Growth:** Appends both the `AssistantMessage` (the request) and the `UserMessage` (the result) to the conversation history.
4. **Re-Inference:** Calls `yield* query(...)` recursively. This sends the *entire updated trajectory* back to the LLM.

### Concurrency & Serial Logic
The system optimizes execution speed by analyzing tool metadata:
- **`runToolsConcurrently`**: Triggered when **all** requested tools in a turn are `isReadOnly()`. It uses `AsyncGenerator` to resolve up to 10 calls in parallel.
- **`runToolsSerially`**: Triggered if **any** tool has side effects (like `FileEditTool`). This prevents race conditions where one tool might attempt to read a file while another is mid-write.

---

## 2. Stateful Execution: `src/utils/PersistentShell.ts`
Claude Code solves the "Lost Shell State" problem by maintaining a long-running child process.

- **Process Bridge:** The `PersistentShell` class spawns a single `/bin/bash` or `/bin/zsh` instance at the start of the session.
- **Environment Persistence:** Because the shell is persistent, variables, aliases, and directory changes (`cd`) carry over between separate `BashTool` calls.
- **Output Streaming via Temp Files:** To avoid pipe buffer overflows and hanging, the system uses unique temporary files in `/tmp/claude-*` to capture `stdout`, `stderr`, and `exit_status`.

---

## 3. The Validation Pipeline
Before any code is written, the `FileEditTool.tsx` executes a strict safety protocol:

1. **Integrity Check:** Verifies the `old_string` exists exactly once in the file.
2. **Freshness Check:** Compares the file's current `mtimeMs` against the `readFileTimestamps` map.
3. **Atomic Write:** Uses `writeFileSync` with `flush: true` to ensure the OS commits the change immediately.

---

## 4. How to Extend This System
- **Adding Post-Processing:** You can hook into the `call()` method of `FileEditTool` to trigger automatic code formatting (e.g., `prettier`) immediately after a successful write.
- **Custom Verifiers:** You can modify the `query` loop to always inject a "Run Linter" tool use if a file edit was detected in the previous turn.

---
