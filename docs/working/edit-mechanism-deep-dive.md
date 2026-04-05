# Architectural Masterclass: The Claude Code Mutation System

This manual provides a high-resolution technical blueprint of the `old_string` / `new_string` exchange. It is designed for engineers who need to understand, maintain, or rebuild the system from scratch.

---

## 1. The Core Data Flow: The "Edit Trajectory"

A code change is never a single event; it is a stateful trajectory involving four distinct entities:
1.  **The Context State:** Global memory of what has been read.
2.  **The LLM (Claude):** The reasoning engine that calculates the delta.
3.  **The CLI Tool (`FileEditTool`):** The validator and disk-writer.
4.  **The Persistent Shell:** The verification environment.

### Data Structure: `readFileTimestamps`
This is the "heartbeat" of the system.
- **Type:** `Record<string, number>` (Maps Absolute Path -> `mtimeMs`).
- **Location:** Managed as a `useRef` in `REPL.tsx` and passed via `ToolUseContext`.
- **Purpose:** To implement **Optimistic Locking** for the local filesystem.

---

## 2. Step-by-Step Implementation Guide

### Phase A: The Observation Turn (Turn 1)
When you ask for a change, Claude first calls `FileReadTool`.
- **CLI Implementation:** `readFileSync` uses `detectFileEncoding(filePath)` to ensure it handles `UTF-8`, `ASCII`, or `UTF-16` correctly.
- **State Capture:** After reading, the CLI **must** update the global timestamp:
  ```typescript
  readFileTimestamps[fullPath] = statSync(fullPath).mtimeMs;
  ```

### Phase B: The Inference (The LLM's Work)
Claude receives the raw text. It doesn't use regex or line numbers. It selects a "Block Anchor."
- **Anchor Logic:** It chooses a 10-15 line block that contains the change.
- **Uniqueness Check:** Claude simulates the search in its own memory. It ensures that the `old_string` it generates does not exist anywhere else in that file.

### Phase C: The Validation Pipeline (The CLI's Enforcement)
When the LLM sends the `FileEditTool` call, the CLI executes these checks in order:

#### 1. Content Integrity (The "Exact Match")
```typescript
const file = readFileSync(fullPath, encoding);
if (!file.includes(old_string)) {
    throw new Error("String to replace not found. This usually happens if there's a space/tab mismatch between the LLM's memory and the disk.");
}
```

#### 2. Ambiguity Check (The "Uniqueness Guard")
```typescript
const matchCount = file.split(old_string).length - 1;
if (matchCount > 1) {
    throw new Error(`Safety Alert: Found ${matchCount} matches. Edit is ambiguous.`);
}
```

#### 3. Stale-Read Detection (The "TOCTOU" Guard)
```typescript
const currentMtime = statSync(fullPath).mtimeMs;
if (currentMtime > readFileTimestamps[fullPath]) {
    throw new Error("File changed on disk by another process. Re-read required.");
}
```

### Phase D: The Atomic Swap
If all checks pass, the CLI performs a literal replacement:
```typescript
const updatedContent = file.replace(old_string, () => new_string);
writeTextContent(fullPath, updatedContent, encoding, lineEndings);
```
- **Nuance:** `writeTextContent` (in `src/utils/file.ts`) re-applies the detected line endings (`LF` vs `CRLF`) to ensure no "Invisible diffs" are created.

---

## 3. Handling Batch & Multi-File Edits

When Claude suggests 10 edits at once:
1.  **Orchestrator (`query.ts`)** sees the array of tools.
2.  It detects **`isReadOnly() === false`** tools in the mix.
3.  It switches to **`runToolsSerially`**.
4.  **Loop:** It processes Tool 1, waits for disk write, updates the timestamp, then moves to Tool 2.
5.  **Sequential Safety:** If Edit 1 and Edit 2 are in the same file, Edit 2's `old_string` must be valid *after* Edit 1 has been applied.

---

## 4. Troubleshooting & Common Failures

| Failure | Cause | Solution |
| :--- | :--- | :--- |
| **"String not found"** | LLM hallucinated a tab vs space or a comment. | CLI reports exactly what it saw to the LLM to trigger a re-read. |
| **"Multiple matches"** | The anchor block was too generic (e.g., just `return true;`). | LLM is prompted to include more surrounding lines (e.g., function signature). |
| **"File modified"** | You hit `Ctrl+S` in VS Code while Claude was thinking. | Claude re-reads the file and re-calculates the diff. |

---

## 5. How to Extend This System
- **Adding Fuzzy Match:** You could modify `validateInput` to use a Levenshtein distance check if an exact match fails by 1-2 characters.
- **Dry-Run Mode:** You could add a flag to `call()` that returns the `structuredPatch` without calling `writeFileSync`.
- **Linting Integration:** You could trigger `BashTool` (eslint) automatically inside the `FileEditTool`'s `call()` function before finalizing the write.

---
