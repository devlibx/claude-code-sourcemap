# Internal Working Documentation: `/compact` Flow

This directory contains the detailed architectural research and flow analysis for the Claude Code command system.

## 🚀 Overview: The `/compact` Command

The `/compact` command is a high-level state transition that performs **Semantic Context Compression**. It allows a developer to continue a long session by summarizing past work and resetting the terminal state without losing the "thread" of the conversation.

---

## 📊 Process Flow (Sequence Diagram)

Below is the end-to-end execution flow from User Input to React State re-hydration.

![Compact Command Flow](./CompactCommandFlow_Detailed.png)

> **Source:** [compact-flow.puml](./compact-flow.puml) (PlantUML)

---

## 🧠 Core Components & Concepts

### 1. [Semantic Summarization](./compact-deep-dive.md#phase-a-context-gathering--summarization)
The system leverages **Claude 3.5 Sonnet** to transform a heavy message history into a concise, actionable summary. This is not just a text summary; it is a strategic "handoff" for the agent's next turn.

### 2. [State Destruction](./compact-deep-dive.md#phase-b-state-destruction-the-wipe)
To free up context and improve performance, the system performs a multi-layer wipe:
- **Terminal UI:** Clears scrollback using ANSI escape codes.
- **Global Store:** Wipes the message history array in `src/messages.ts`.
- **Logic Caches:** Invalidates memoized directory and git status maps.

### 3. [React Re-hydration](./compact-deep-dive.md#phase-c-state-re-hydration-the-baton-pass)
Using a "Baton Pass" pattern, the command logic queues the summary in `setForkConvoWithMessagesOnTheNextRender`. The main `REPL.tsx` component then "picks up" this summary and restores it as the new foundational history.

---

## 📂 Files in this Directory

| File | Description |
| :--- | :--- |
| [README.md](./README.md) | This index file. |
| [compact-deep-dive.md](./compact-deep-dive.md) | Deep technical analysis of the compression algorithm. |
| [compact-flow.puml](./compact-flow.puml) | PlantUML source for the sequence diagram. |
| [CompactCommandFlow_Detailed.png](./CompactCommandFlow_Detailed.png) | Rendered sequence diagram (Handwritten style). |

---
*Created by Senior Architect Gemini CLI for deep-dive training purposes.*
