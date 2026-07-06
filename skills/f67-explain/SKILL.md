---
name: f67-explain
description: >
  Explains architecture, current implementation, business flow, dependencies, or data flow
  for any part of the project using F67 memory and graphs — read-only, no modifications.
  Trigger with "/f67-explain [topic]", "explain how X works in this project", or
  "walk me through the flow of".
---

# /f67-explain — Explain the system

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Pipeline

1. Classify the topic in-session from `memory/index.json`, then dispatch `f67-memory-loader` for the matched domains.
2. If memory fully answers the question, answer from memory and say so.
3. Otherwise dispatch `f67-discovery` to fill the gaps from code, then explain.

## Explanation quality

- Anchor every claim in a source: memory file, decision record, or `path:line`.
- For flows, show the chain: entry point → services → persistence → events, with paths.
- Distinguish business rules (why) from implementation (how).
- Note known issues and historical context from `history.md`/`decisions.md` where relevant — that is what makes F67 explanations better than a cold repo read.
- If memory contradicted code during discovery, mention it and suggest `/f67-sync`.

Read-only: never modify files, memory, or state.
