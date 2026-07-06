---
name: f67-review
description: >
  Performs a complete F67 code review of the current workflow's changes — architecture,
  security, performance, maintainability, consistency, accessibility — producing a
  review-report artifact with a verdict. Trigger with "/f67-review", "review the current
  f67 work", or "run the f67 review workflow".
---

# /f67-review — Review the work

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Preconditions

`state/changed-files.json` must be non-empty for the active artifact. Warn (but proceed if the user confirms) when tasks remain `pending` in the plan or `/f67-test` has not run.

## Pipeline

1. Dispatch `f67-reviewer` with the artifact folder path. It writes `review-report.md` with located, severity-ranked findings and a verdict.
2. User report — headlines only: verdict, blockers/majors one line each, counts of minors/nits, artifact path. Nothing else unless asked.
3. Route by verdict: `approve` → dispatch `f67-memory-evolver` in delta mode now (mandatory — memory freshness does not wait for `/f67-sync`); `approve-with-fixes` / `rework` → suggest `/f67-improve`, after which the evolver runs.
4. Append the metrics line to `logs/metrics.jsonl`, including the report's finding categories — the evolver uses recurrence to turn repeated criticism into conventions.

The orchestrator never reviews code itself and never edits code in this workflow.
