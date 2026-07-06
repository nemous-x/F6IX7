---
name: f67-test
description: >
  Runs the F67 testing workflow — generates missing tests for completed tasks, suggests
  edge cases, runs suites, and validates against regressions. Trigger with "/f67-test",
  "test the current f67 work", or "generate tests for the current plan".
---

# /f67-test — Test the current work

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Preconditions

An active artifact folder with `execution-report.md` and entries in `state/changed-files.json`. If nothing has been implemented, offer to test a user-specified area instead (dispatch discovery first to scope it).

## Pipeline

1. Dispatch `f67-tester` with the artifact folder path. It maps acceptance criteria to coverage, writes missing tests, runs the affected and regression suites, and appends `## Testing` to the execution report.
2. If multiple completed tasks are independent, dispatch one `f67-tester` per task in parallel.
3. User report — conclusions only: suites run + results, tests added, defects found. No test code.
4. If defects were found, offer `/f67-implement` (as a fix task — dispatch `f67-task-decomposer` to add fix tasks to the plan) rather than letting the tester patch implementation code.

Suggest `/f67-review` once tests are green.
