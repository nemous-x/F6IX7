---
name: f67-improve
description: >
  Turns F67 review findings into a prioritized improvement plan and optionally applies
  approved improvements one at a time. Trigger with "/f67-improve", "fix the review
  findings", or "apply the f67 improvements".
---

# /f67-improve — Plan and apply improvements

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Preconditions

`review-report.md` must exist in the active artifact folder; otherwise suggest `/f67-review` first.

## Pipeline

1. Dispatch `f67-improver` in plan-only mode. It writes `improvement-plan.md` (must-fix / should-fix / deferred-as-debt).
2. Present the plan and ask the user which items to apply (default: all must-fix items).
3. Dispatch `f67-improver` in apply mode with the approved item list. It applies items one at a time, testing after each, and updates the execution report.
4. If blockers were fixed, suggest a focused re-review (`/f67-review`).
5. **Workflow-end memory (mandatory)**: once application finishes (or the user declines to apply), dispatch `f67-memory-evolver` in delta mode to fold the workflow into memory and the index.
6. Append the metrics line to `logs/metrics.jsonl`.

Deferred items are recorded for the memory evolver to write into the domain's `known-issues.md`. Never apply items the user did not approve.
