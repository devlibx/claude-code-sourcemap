# In-Depth Technical Deep Dive: The `/compact` Command

This document provides a senior-level architectural breakdown of the `/compact` command in Claude Code. It explains how a massive conversation is lossily compressed into a summary while maintaining agentic continuity.

## 1. Architectural Overview
The `/compact` command is a **State Transition Mechanism**. Its goal is to move the conversation from a high-token, high-latency state ($S_{full}$) to a low-token, high-focus state ($S_{compact}$).

## 2. Step-by-Step Flow Analysis

### Phase A: Context Gathering & Summarization
*   **Location:** `src/commands/compact.ts`
*   **Logic:** The command retrieves the current message history from the global getter in `src/messages.ts`.
*   **The Prompt:** It appends a specialized `UserMessage` requesting a summary. 
*   **The Model:** It uses `querySonnet` (the "capable" model) specifically because summarization of complex technical work requires high global coherence. Using a smaller model (like Haiku) would risk losing critical context.

### Phase B: State Destruction (The "Wipe")
*   **Terminal Reset:** Calls `clearTerminal()` which uses the ANSI sequence `\x1b[2J\x1b[3J\x1b[H`. Note the `3J` specifically, which wipes the scrollback buffer.
*   **Memory Reset:** Directly calls `getMessagesSetter()([])`. This clears the internal array that stores the conversation. At this point, the application "knows" nothing about the past.
*   **Cache Invalidation:** Calls `getContext.cache.clear()`. This is vital. It forces Claude to re-index the project directory and git status, ensuring the "new" conversation starts with up-to-date project maps.

### Phase C: State Re-hydration (The "Baton Pass")
This is the most sophisticated part of the implementation. It uses a **delayed state update pattern** to move data from the Command Logic into the React UI.

1.  **Queueing:** `compact.ts` calls `setForkConvoWithMessagesOnTheNextRender([instr, summary])`. This updates a state variable in the `REPL.tsx` component.
2.  **Triggering:** Because this is a React state update, it triggers a re-render.
3.  **Handoff Hook:** `src/screens/REPL.tsx` has a `useEffect` that watches this specific "fork" variable.
    ```typescript
    useEffect(() => {
      if (forkConvoWithMessagesOnTheNextRender) {
        setMessages(forkConvoWithMessagesOnTheNextRender); // Memory is RESTORED with just the summary
        setForkConvoWithMessagesOnTheNextRender(null); // Clear the trigger
      }
    }, [forkConvoWithMessagesOnTheNextRender]);
    ```

## 3. The "AI Memory" Magic
How does Claude actually "remember" the summary next time?

When the user types their *next* question, the system calls `query` in `src/query.ts`. This function gathers the current `messages` array. Because of the handoff above, the array looks like this:
1.  **User Turn:** "Use the /compact command to clear history..."
2.  **Assistant Turn:** "[The Summary Content from Sonnet]"
3.  **User Turn:** "[Your next question]"

When this is sent to the API, the model sees the summary as its **own previous response**. This is the most weighted part of the context window, effectively seeding the model's "Short-Term Memory" with the "Long-Term Summary" of the previous session.

## 4. Why This Pattern?
*   **Cost Efficiency:** Drastically reduces input token costs for long-running sessions.
*   **Focus Restoration:** Eliminates "distractions" or hallucinations caused by stale tool results or irrelevant side-conversations from earlier in the day.
*   **Performance:** Faster turn-around times as the API doesn't have to process thousands of tokens of history.

---
*Created by Senior Architect Gemini CLI for deep-dive training purposes.*
