---
name: f67-docs
description: >
  Generates or updates project documentation from F67 memory and completed workflow
  artifacts — READMEs, API docs, architecture docs, domain guides. Trigger with
  "/f67-docs", "/f67-docs [domain or area]", "update the docs for", or "document the
  billing domain".
---

# /f67-docs — Generate or update documentation

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Scope selection

- With an argument: document that domain/area.
- Without: infer from the active artifact (document what the completed workflow changed); if no active workflow, ask what to document.

## Pipeline

1. Dispatch `f67-memory-loader` for the scope — memory is the primary documentation source (that is the point of DDM: docs describe intent and rules, not just code).
2. Dispatch `f67-discovery` only for details memory lacks (signatures, exact routes, config keys).
3. Write or update the documentation yourself in the project's existing docs location and style (check for docs/, README conventions, docusaurus/mkdocs config). Match the project's voice and formatting.
4. Cover: purpose, business rules, architecture, API surface, data model, how to test, known issues — proportional to the audience of the doc being written.

## Rules

- Update existing docs in place; do not create parallel documents.
- Facts only from memory or code — flag anything unverifiable instead of guessing.
- Offer to record significant documentation decisions in memory via the evolver.
