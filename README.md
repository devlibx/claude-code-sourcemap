# Internal Working Documentation: Claude Code Flows

This repository contains detailed architectural research and flow analysis for the Claude Code agentic system. It is designed to be a "Self-Service Engineering Portal" for anyone looking to understand or extend the codebase.

---

## 🚀 1. The `/compact` Command Flow
**Concept:** Semantic Context Compression to manage long sessions.

- **[Deep Dive](./docs/working/compact-deep-dive.md):** Detailed analysis of how high-token conversations are condensed using Claude 3.5 Sonnet without losing agentic intent.
- **[Flow Diagram](./docs/working/compact-flow.puml):** Maps the "Baton Pass" from the command logic to the React REPL re-hydration cycle.

![Compact Flow](./docs/working/CompactCommandFlow_Detailed.png)

---

## 🛠️ 2. The Code Editing Loop
**Concept:** The recursive "Read-Think-Edit-Verify" cycle.

- **[Deep Dive](./docs/working/code-editing-deep-dive.md):** Breakdown of the `query.ts` recursive generator, including concurrency management (`Parallel Reads` vs `Serial Writes`).
- **[Flow Diagram](./docs/working/code-editing-flow.puml):** Multi-turn interaction between the CLI, the LLM, and the Persistent Shell environment.

![Code Editing Flow](./docs/working/CodeEditingFlow_Expert.png)

---

## 🛡️ 3. Mutation Safety (`old_string` / `new_string`)
**Concept:** Exact-match string replacement to prevent code corruption.

- **[The Master Manual](./docs/working/edit-mechanism-deep-dive.md):** **START HERE** if you want to understand the code-writing engine. Includes:
    - **Implementation Guide:** Step-by-step logic for Junior Engineers.
    - **Validation Pipeline:** Detailed breakdown of the Content, Uniqueness, and Stale-Read checks.
    - **Extension Guide:** How to add new features like fuzzy-matching or auto-linting.
- **[Flow Diagram](./docs/working/edit-mechanism-flow.puml):** Expert-level sequence diagram showing multi-file batch processing and JSON tool-use payloads.

![Edit Mechanism Flow](./docs/working/EditMechanismFlow_Expert.png)

---

## 📂 Documentation Manifest

| File | Category | Description |
| :--- | :--- | :--- |
| [compact-deep-dive.md](./docs/working/compact-deep-dive.md) | Compact | Context compression & session reset logic. |
| [compact-flow.puml](./docs/working/compact-flow.puml) | Compact | PUML source for state re-hydration. |
| [code-editing-deep-dive.md](./docs/working/code-editing-deep-dive.md) | Editing | Recursive tool-use & trajectory management. |
| [code-editing-flow.puml](./docs/working/code-editing-flow.puml) | Editing | PUML source for the agentic loop. |
| [edit-mechanism-deep-dive.md](./docs/working/edit-mechanism-deep-dive.md) | Mutation | **Technical Blueprint** for the string-swap engine. |
| [edit-mechanism-flow.puml](./docs/working/edit-mechanism-flow.puml) | Mutation | PUML source for batch atomic writes. |

---
