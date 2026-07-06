---
name: f67-prompt
description: >
  Starts the F67 prompt-engineering workflow — transforms a user request into a structured
  execution specification (prompt-spec.md) via domain detection, memory loading, discovery,
  and context building. Never writes code. Trigger with "/f67-prompt [request]",
  "create an f67 spec for", or "run the f67 prompt workflow".
---

# /f67-prompt — Build an execution specification

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md` for pipeline and artifact contracts. The argument is the user's raw request. If `.claude/f67/` is missing, tell the user to run `/f67-init` first.

## Pipeline

1. **Classify in-session** — read `memory/index.json` (one small file) and classify the request yourself: primary/secondary domains (keywords + summaries), technical areas, complexity, greenfield. Record in `state/current-session.json`. No detection dispatch.
2. **Freshness check** — compare index `lastSyncCommit` with `git rev-parse HEAD`. If the repo has moved substantially since memory last synced, say so in one line; for `large` requests run the sync delta first — a spec built on stale memory is a wrong spec.
3. **Route by complexity**: `trivial`/`small` → tell the user this fits `/f67-execute` and offer it (one line). If they want a spec anyway, or complexity is `medium`+, continue.
4. Create the artifact folder `.claude/f67/artifacts/<NNN>-<slug>/`; record it in `state/current-session.json`.
5. **Parallel dispatch** — `f67-memory-loader` and `f67-discovery` in the same message: the loader gets your classification; discovery starts from the domains' `related-files.json` (it does not wait for the digest).
6. `medium`: skip the context-builder — dispatch `f67-prompt-builder` directly with both digests. `large`: dispatch `f67-context-builder` first, then `f67-prompt-builder`.
7. The prompt-builder writes `prompt-spec.md` and updates `state/current-spec.json`.

## Orchestrator discipline

- Pass artifact paths and compact JSON between agents — never paste full reports into your own context.
- If the prompt-builder surfaces blocking open questions, relay only the questions (no recap), collect answers, re-dispatch with answers appended.
- Final user report — conclusions only: objective, domains, criteria count, open questions if any, artifact path, next step. No spec restatement, no process narration.
- Append the metrics line to `logs/metrics.jsonl` (see core conventions).

Never plan, decompose, or implement in this workflow.
