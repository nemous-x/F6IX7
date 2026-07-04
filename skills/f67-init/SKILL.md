---
name: f67-init
description: >
  Initializes F67 for an existing repository — analyzes the codebase, detects stack,
  architecture, conventions, testing and UI frameworks, identifies business domains,
  and generates the initial Domain Driven Memory and relationship graphs under
  .claude/f67/. Trigger with "/f67-init", "initialize f67", "onboard this repo to f67",
  or "set up f67 memory".
---

# /f67-init — Initialize F67 for an existing project

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md` first — it defines the runtime layout being created. Initialization is repeatable and incremental: if `.claude/f67/` already exists, update rather than recreate, and never touch curated content.

## Phase 1 — Repository analysis (dispatch agents; stay light)

Dispatch the `f67-discovery` agent (or multiple in parallel for large repos) to determine:

1. Frameworks and stack (read package.json / pyproject / go.mod etc., lockfiles, build config).
2. Architecture style and patterns (layering, module boundaries, DDD/hexagonal/MVC, monorepo layout).
3. Coding conventions (linter/formatter configs, naming in practice, error-handling style).
4. Project guidance files: CLAUDE.md, AGENTS.md, README, CONTRIBUTING, docs/ — extract binding rules.
5. Testing framework, test placement, fixture patterns, coverage expectations.
6. UI framework, component library, styling approach, design tokens/design system.

## Phase 2 — Domain identification

From folder structure, module names, route groups, database schema, and domain language in code, propose the business domain list (e.g. authentication, billing, orders). Present the proposed domains to the user for confirmation or adjustment before generating memory — wrong domain boundaries poison everything downstream.

## Phase 3 — Generate DDM

Create `.claude/f67/` per the core conventions layout:

1. `config.yaml` — fill `project.*` from Phase 1; set `skills.enabled` from detected stack.
2. `memory/global/` — write architecture.md, coding-standards.md, conventions.md, testing.md, ui.md, design-system.md, security.md, glossary.md from Phase 1 findings. Use templates in `${CLAUDE_PLUGIN_ROOT}/templates/memory/global/`. Only create files with real content.
3. `memory/domains/<domain>/` — for each confirmed domain, dispatch `f67-discovery` scoped to that domain, then write its memory files from `${CLAUDE_PLUGIN_ROOT}/templates/memory/domain/`. Populate related-files.json and per-domain graph.json.
4. `graphs/` — build domain-graph.json, file-graph.json, dependency-graph.json per `${CLAUDE_PLUGIN_ROOT}/schemas/graph.schema.json`.
5. `state/` — initialize empty state files from `${CLAUDE_PLUGIN_ROOT}/templates/state/`.

For large repos, process domains sequentially and checkpoint progress in `state/progress.json` so init can resume.

## Phase 4 — Validate

Before declaring init complete:

- Every domain has overview.md and related-files.json with existing paths.
- Graphs parse against the schema; all edges reference existing nodes.
- Spot-check 3 random file→domain associations by reading the files.
- Report a summary: domains created, files mapped, coverage gaps.

## Phase 5 — Recommendations

After validation, always close with tailored recommendations:

1. **Business-specific skills**: for each confirmed domain with substantial business rules, suggest the user create a project skill in `.claude/f67/skills/<domain>.md` (same format as the plugin's stack skills) encoding that domain's non-obvious rules and patterns. Offer to draft them from the generated memory.
2. **Architecture for young projects**: if the repository is new or largely empty, recommend adopting DDD (domain/backend-heavy products) or feature-sliced architecture (frontend-heavy products) before feature work begins, explain the tradeoff in two sentences each, and record the user's choice as the first decision record.
3. **TDD for business logic**: note that F67's planner defaults to TDD for business-logic features and where that expectation comes from.

Recommend `/f67-sync` cadence and `/f67-prompt` as the next step.
