---
name: f67-plan
description: >
  Generates the F67 implementation plan from the current prompt-spec — strategy, milestones,
  risks, rollback, testing strategy, and a decomposed task tree of small executable tasks.
  Trigger with "/f67-plan", "plan the current f67 spec", or "create the f67 task tree".
---

# /f67-plan — Plan and decompose

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Preconditions

`state/current-spec.json` must point to a prompt-spec. If absent, run the `/f67-prompt` workflow first (ask the user for the request if none was given). If the spec has unresolved blocking questions, surface them and stop.

## Pipeline

1. `medium` complexity (from `state/current-session.json`): dispatch `f67-planner` alone with instructions to also produce the task tree in the same pass — one dispatch instead of two. `large`: dispatch `f67-planner`, then `f67-task-decomposer`.
2. Sanity-check the result yourself (orchestrator-level, no code reading): every acceptance criterion owned by a task, dependencies form a DAG, no task touches more than ~5 files.
3. Mark tasks that share no files and no dependency edge as parallelizable, so implement/test phases can run them concurrently.
4. Append the metrics line to `logs/metrics.jsonl`.

## Report to the user — conclusions only

Strategy with its reason, milestones, task count + first task, top risks, artifact path, next step. No plan restatement.

Never implement anything in this workflow.
