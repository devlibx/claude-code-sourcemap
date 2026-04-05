# Architectural Masterclass: The `/compact` Command

This manual provides a high-resolution technical blueprint of the Context Compression and State Re-hydration system. It is designed for engineers who need to understand how Claude Code maintains a "perfect memory" while staying within budget and token limits.

---

## 1. The Core Problem: Context Saturation
As an agentic session progresses, the conversation history grows linearly. 
- **The Cost:** Input tokens become increasingly expensive.
- **The Latency:** The LLM takes longer to process the full history.
- **The Noise:** Stale tool results and old code blocks "distract" the model's attention.

The `/compact` command is a **Lossy Compression Algorithm** for the session state.

---

## 2. Step-by-Step Implementation Guide

### Phase A: The Semantic Summarization
The command doesn't just "delete" history; it translates it.
1.  **History Retrieval:** Calls `getMessagesGetter()()` to fetch the full array of `Message` objects.
2.  **Request Construction:** Appends a hidden `UserMessage` with a high-fidelity prompt:
    > *"Focus on what we did, what we're doing, which files we're working on, and what we're going to do next."*
3.  **The LLM Turn:** It uses `querySonnet` (the heavy-reasoning model) to perform the compression.
    - **Note:** It bypasses standard UI hooks to keep this "internal" turn invisible to the user.

### Phase B: The System Wipe (Synchronous)
Once the summary is received, the CLI performs a multi-layer reset:
1.  **Memory Wipe:** `getMessagesSetter()([])` - Directly empties the global message array.
2.  **Terminal Reset:** `clearTerminal()` - Sends ANSI `\x1b[2J\x1b[3J\x1b[H` to clear screen and scrollback buffer.
3.  **Cache Invalidation:** `getContext.cache.clear()` - Forces Claude to re-scan the project (README, Git status, etc.) during the next turn.

### Phase C: State Re-hydration (The Baton Pass)
This is the most critical React pattern in the codebase. We cannot simply "set messages" because the command is running inside a logic handler, not a component.

1.  **Queueing:** The command calls `setForkConvoWithMessagesOnTheNextRender([instruction, summaryResponse])`.
2.  **State Handoff:**
    - In `REPL.tsx`, a `useEffect` hook watches this variable.
    - When it detects data, it increments the `forkNumber` (creating a new session scope).
    - It then calls its local `setMessages(data)` to officially "seed" the new conversation.

---

## 3. The "Token Mocking" Hack: UI Engineering
Claude Code has a `TokenWarning` component that calculates costs based on the **last assistant message's usage data**.

If we just injected the summary, the UI would still show high token usage because the summary itself was generated from a large context. To fix this, `compact.ts` manually overwrites the metadata:

```typescript
summaryResponse.message.usage = {
  input_tokens: 0, // <--- THE HACK
  output_tokens: summaryResponse.message.usage.output_tokens,
  cache_creation_input_tokens: 0,
  cache_read_input_tokens: 0,
}
```
**Why?** This "tricks" the UI into acknowledging that the context window is now effectively "fresh" and empty, even though the summary text is present.

---

## 4. How to Extend This System
- **Adding Auto-Compact:** You could modify `src/query.ts` to trigger the `/compact` logic automatically once the `input_tokens` exceeds a certain threshold (e.g., 150k tokens).
- **Persistent Checkpoints:** You could modify the handoff to save the summary to a `.claude/checkpoints/` directory, allowing users to "resume" a summarized session later.
- **Custom Summary Styles:** Add a flag like `/compact --short` or `/compact --detailed` to change the prompt sent to Sonnet.

---

## 5. Troubleshooting & Edge Cases
- **Failure to Summarize:** If the API fails during summarization, the command throws an error and **does not** wipe the history. This ensures the user never loses data due to a network error.
- **Lost Context:** If a user runs `/compact` while a tool is mid-execution, the `abortController` ensures the background process is terminated before the reset.

---
