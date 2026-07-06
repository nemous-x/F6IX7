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
2. User report — conclusions only: verdict, blockers/majors, counts of minors/nits, artifact path.
3. Route by verdict:
   - `approve` → suggest `/f67-improve` (optional) then let the memory evolver run (see `/f67-sync` or workflow completion).
   - `approve-with-fixes` / `rework` → suggest `/f67-improve` to plan the fixes.

3. **Workflow-end memory (mandatory)**: on `approve`, dispatch `f67-memory-evolver` in delta mode now — memory freshness does not wait for `/f67-sync`. On other verdicts, the evolver runs after `/f67-improve` completes.
4. Append the metrics line to `logs/metrics.jsonl`.

The orchestrator never reviews code itself and never edits code in this workflow.
