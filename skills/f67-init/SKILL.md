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

Dispatch `f67-discovery` agents in parallel (one per concern below, or per app in a monorepo) to determine:

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

1. `config.yaml` — fill `project.*` from Phase 1. Then resolve skills YOURSELF (only the orchestrator sees installed skills — never delegate this): match each technical area the project needs against project skills in `.claude/f67/skills/` first, then installed Claude Code skills/plugins, per `${CLAUDE_PLUGIN_ROOT}/templates/skill-injection-rules.md`. Write the result to `skills.map` (area → path or skill name) and unmatched areas to `skills.requested`. Every future request injects via this map — a lookup, not a search.
2. `memory/global/` — write architecture.md, coding-standards.md, conventions.md, testing.md, ui.md, design-system.md, security.md, glossary.md from Phase 1 findings. Use templates in `${CLAUDE_PLUGIN_ROOT}/templates/memory/global/`. Only create files with real content.
3. `memory/domains/<domain>/` — dispatch `f67-discovery` agents for confirmed domains **in parallel batches**, then write each domain's layer-organized memory (`overview.md`, `business-logic.md`, `backend.md`, `web.md`, `mobile.md` as applicable, `tests.md`) from `${CLAUDE_PLUGIN_ROOT}/templates/memory/domain/`. Populate layer-split related-files.json and per-domain graph.json. Every file dense and ≤150 lines.
4. `graphs/` — build domain-graph.json and dependency-graph.json per `${CLAUDE_PLUGIN_ROOT}/schemas/graph.schema.json` (per-file mapping lives in each domain's related-files.json).
4b. `memory/index.json` — the entry point every command reads first: per-domain summary, keywords, layers, updatedAt, plus lastSyncCommit = current HEAD.
5. `state/` — initialize empty state files from `${CLAUDE_PLUGIN_ROOT}/templates/state/`.

For large repos, process domains in parallel batches and checkpoint progress in `state/progress.json` after each batch so init can resume.

## Phase 4 — Validate

Before declaring init complete:

- `memory/index.json` exists, parses, and lists every domain with keywords and layers.
- Every domain has overview.md and related-files.json with existing paths.
- Graphs parse against the schema; all edges reference existing nodes.
- Spot-check 3 random file→domain associations by reading the files.
- Report a summary: domains created, files mapped, coverage gaps.

## Phase 5 — Recommendations

After validation, always close with tailored recommendations:

1. **Skill requests**: present `skills.requested` — for each detected stack/requirement with no matching installed skill, recommend installing one from a plugin marketplace or offer to generate a project skill draft from the repo's own conventions (per skill-injection-rules.md, evidence-based, marked `f67:generated`).
2. **Business-specific skills**: for each confirmed domain with substantial business rules, suggest the user create a project skill in `.claude/f67/skills/<domain>.md` encoding that domain's non-obvious rules and patterns. Offer to draft them from the generated memory.
3. **Architecture for young projects**: if the repository is new or largely empty, recommend adopting DDD (domain/backend-heavy products) or feature-sliced architecture (frontend-heavy products) before feature work begins, explain the tradeoff in two sentences each, and record the user's choice as the first decision record.
4. **TDD for business logic**: note that F67's planner defaults to TDD for business-logic features and where that expectation comes from.

Recommend `/f67-sync` cadence and `/f67-prompt` as the next step.
