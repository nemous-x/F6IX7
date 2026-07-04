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

1. Create the artifact folder `.claude/f67/artifacts/<NNN>-<slug>/` (next sequence number, slug from the request). Record it in `state/current-session.json`.
2. Dispatch `f67-domain-detector` with the request. Save its JSON to `state/selected-domains.json`.
3. Dispatch `f67-memory-loader` with the detection JSON. Pass its digest forward.
4. Dispatch `f67-discovery` with the digest's gaps and related files.
5. Dispatch `f67-context-builder` to merge everything into `context.md` in the artifact folder.
6. Dispatch `f67-prompt-builder` to write `prompt-spec.md` and update `state/current-spec.json`.

## Orchestrator discipline

- Pass artifact paths and compact JSON between agents — never paste full reports into your own context.
- If the prompt-builder surfaces blocking open questions, relay them to the user, collect answers, and re-dispatch the prompt-builder with the answers appended.
- Finish by summarizing the spec in a few sentences and pointing at the artifact path. Suggest `/f67-plan` as the next step.

Never plan, decompose, or implement in this workflow.
