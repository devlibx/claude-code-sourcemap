# Architectural Deep Dive: The Code Editing & Agentic Loop

This document provides a senior-level technical analysis of the recursive orchestration engine behind Claude Code. It explains how the system manages state, concurrency, and safety during autonomous code modifications.

---

## 1. The Orchestration Engine: `src/query.ts`
At the heart of Claude Code is a **Recursive Generator Function**. Unlike a traditional request-response model, the `query` function manages a "trajectory" of multiple turns.

### Concurrency Strategy
The system optimizes execution speed by analyzing tool metadata before dispatching calls:
- **`runToolsConcurrently`**: Triggered when **all** requested tools are `isReadOnly()`. The system uses `AsyncGenerator` to resolve these in parallel, significantly reducing discovery time.
- **`runToolsSerially`**: Triggered if **any** tool has side effects (like `FileEditTool`). This prevents race conditions where one tool might attempt to read a file while another is mid-write.

### Recursive Re-Inference
The loop terminates only when an `AssistantMessage` contains **zero** `tool_use` blocks. If tools are used, the system automatically:
1.  Collects all `tool_result` blocks.
2.  Appends them to the current conversation history.
3.  Calls `query()` again, effectively asking the LLM: *"Here is what the tools returned. What is your next move?"*

---

## 2. Stateful Execution: `src/utils/PersistentShell.ts`
A common failure point for AI agents is losing shell state (e.g., `cd` into a directory and losing it in the next turn). Claude Code solves this with a **Persistent Shell Process**.

- **Single Child Process:** The `PersistentShell` class spawns a long-running `/bin/bash` or `/bin/zsh` instance.
- **State Persistence:** Environment variables (like `VIRTUAL_ENV`), aliases, and current working directories (`cwd`) are maintained across multiple `BashTool` calls.
- **Bridge via Temp Files:** Because standard pipes can hang, the system uses unique temporary files (stored in `/tmp/claude-*`) to capture `stdout`, `stderr`, and `exit_status` for every command execution.

---

## 3. Multi-Layered Mutation Safety
The `FileEditTool` is designed with a "Fail-Safe" philosophy to protect the developer's source of truth.

### The "Stale Write" Protection
To prevent the agent from overwriting manual changes made by the human, the system uses a **Time-of-Check to Time-of-Use (TOCTOU)** guard:
1.  **Ingestion:** When `FileReadTool` reads a file, it stores the file's `mtimeMs` (last modified time) in a global `readFileTimestamps` map.
2.  **Validation:** Before `FileEditTool` writes to the file, it re-fetches the current `mtimeMs` from the disk.
3.  **Conflict Resolution:** If the disk's timestamp is newer than the `readFileTimestamps` record, the tool **aborts**. It forces the LLM to re-read the file before attempting another edit.

### The Uniqueness Constraint
Claude Code uses **Exact String Replacement** rather than AST-based editing. This is more robust against syntax errors but prone to "wrong-location" edits.
- **Guard:** The `validateInput` function counts the occurrences of `old_string`. 
- **Rule:** If the count is anything other than exactly **1**, the tool fails. This forces the LLM to provide more context lines (usually 3-5 lines above and below) to uniquely identify the change site.

---

## 4. The Loop Lifecycle: A Worked Example
**Task:** *"Fix the bug in app.ts and run tests."*

1.  **Turn 1 (Read):** LLM calls `FileReadTool`. System records `mtimeMs`.
2.  **Turn 2 (Think):** LLM analyzes code, plans a fix.
3.  **Turn 3 (Write):** LLM calls `FileEditTool`. System verifies `mtimeMs`, performs `.replace(old, new)`, and writes to disk.
4.  **Turn 4 (Verify):** LLM calls `BashTool` to run `npm test`. The persistent shell executes the tests.
5.  **Turn 5 (Close):** If tests pass, the LLM provides a final text response. If they fail, the loop returns to **Step 2**.

---
*Created by Senior Architect Gemini CLI for expert-level training.*
