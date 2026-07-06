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

## Pipeline (dispatch each stage as its agent; never do the work inline)

1. Dispatch `f67-domain-detector` (haiku — fast) with the request. Save its JSON to `state/selected-domains.json`.
2. **Route by complexity**: `trivial`/`small` → tell the user this fits `/f67-execute` and offer it (one line). If they want a spec anyway, or complexity is `medium`+, continue.
3. Create the artifact folder `.claude/f67/artifacts/<NNN>-<slug>/`; record it in `state/current-session.json`.
4. **Parallel dispatch** — `f67-memory-loader` and `f67-discovery` in the same message: the loader gets the detection JSON; discovery starts from `related-files.json` + graphs for the detected domains (it does not wait for the digest). Both return capped digests (100/80 lines).
5. `medium`: skip the context-builder — dispatch `f67-prompt-builder` directly with both digests; it writes `context.md`-level facts straight into the spec's sections. `large`: dispatch `f67-context-builder` first, then `f67-prompt-builder`.
6. The prompt-builder writes `prompt-spec.md` (≤120 lines) and updates `state/current-spec.json`.

## Orchestrator discipline

- Pass artifact paths and compact JSON between agents — never paste full reports into your own context.
- If the prompt-builder surfaces blocking open questions, relay only the questions (no recap), collect answers, re-dispatch with answers appended.
- Final user report — max 10 lines: objective (1 line), domains, acceptance criteria count, open questions if any, artifact path, "next: /f67-plan". No spec restatement.

Never plan, decompose, or implement in this workflow.
