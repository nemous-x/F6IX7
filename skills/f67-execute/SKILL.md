---
name: f67-execute
description: >
  F67 fast path — for short, well-scoped tasks it gathers context and memory quickly and
  executes end to end in a single agent dispatch: context, implement, verify, memory update.
  Trigger with "/f67-execute [task]", "quick f67 task", or "just do this small change with f67".
---

# /f67-execute — Fast path for short tasks

Act as the F67 orchestrator. This is the speed-optimized route: one agent dispatch, no artifact pipeline. Requires `.claude/f67/` (else point to `/f67-init`).

## Flow

1. Dispatch `f67-executor` with the user's request verbatim — session model, full quality. No classification dispatch, no memory-loader, no separate discovery: the executor scopes itself from `memory/index.json` and layer memory.
2. If the executor answers "too large for fast path", relay its one-line reason and offer `/f67-prompt` — do not retry or decompose here.
3. Relay the executor's report as-is — it is already conclusions-only. Add nothing.

## Rules

- Never use this route for multi-domain work, business-rule changes, or migrations — the executor's scope guard enforces this, respect it.
- The executor updates memory itself; do not dispatch the memory evolver afterward.
- Orchestrator output to the user: conclusions only, straight to the point.
- Append the metrics line to `logs/metrics.jsonl`.
