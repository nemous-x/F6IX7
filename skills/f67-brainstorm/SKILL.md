---
name: f67-brainstorm
description: >
  Generates approaches, alternatives, tradeoffs, and ideas for a request using F67 domain
  memory — strictly no code and no repository modifications. Trigger with
  "/f67-brainstorm [topic]", "brainstorm approaches for", or "what are my options for".
---

# /f67-brainstorm — Explore approaches

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Pipeline

1. Dispatch `f67-domain-detector`, then `f67-memory-loader` for the topic — brainstorming must be grounded in the project's actual architecture, constraints, and history (especially past decisions in `memory/decisions/`).
2. Optionally dispatch `f67-discovery` if the topic hinges on how something is currently built.
3. Generate 2–4 distinct approaches yourself. For each: summary, how it fits the existing architecture, tradeoffs (complexity, risk, performance, migration cost), impact on affected domains, rough effort.
4. Compare against any relevant past decisions — flag approaches that would reverse a recorded decision and say why that decision existed.
5. Recommend one approach with reasoning, but present all fairly.
6. If the detection JSON says `greenfield: true` (nothing like this exists yet, or the project is new), explicitly recommend structuring the work with DDD (domain/backend-heavy) or feature-sliced architecture (frontend-heavy), and recommend a TDD strategy if the capability is business-logic-centric.

## Output

Present the comparison in conversation. If the user picks a direction, offer `/f67-prompt` to turn it into a spec. Optionally save the brainstorm as `brainstorm.md` in an artifact folder if the user wants it kept.

Absolute rule: no code, no pseudocode beyond one-line sketches, no file modifications.
