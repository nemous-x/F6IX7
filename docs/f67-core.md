# F67 core conventions

This document is the single source of truth for F67 runtime layout, workflow pipeline, and artifact contracts. Every F67 skill and agent must follow it.

## Runtime layout (inside the target repository)

All F67 project knowledge lives in the repository being worked on, never in the plugin:

```
.claude/f67/
├── config.yaml                 # Project-level F67 configuration
├── memory/
│   ├── global/                 # architecture.md, coding-standards.md, security.md,
│   │                           # testing.md, ui.md, design-system.md, glossary.md, conventions.md
│   ├── domains/<domain>/       # One folder per business domain (see Domain memory files)
│   ├── features/<feature>/     # feature.md + graph.json per completed feature
│   ├── decisions/              # ADR-style decision records (NNNN-title.md)
│   └── sessions/               # Session summaries (YYYY-MM-DD-slug.md)
├── graphs/
│   ├── domain-graph.json       # Domain ↔ domain relationships
│   ├── file-graph.json         # File → domain/feature associations
│   └── dependency-graph.json   # Service/repository/component/API/test edges
├── state/
│   ├── current-session.json
│   ├── active-context.json
│   ├── current-spec.json       # Pointer to active prompt-spec artifact
│   ├── current-plan.json       # Pointer to active plan + task tree
│   ├── progress.json
│   ├── changed-files.json
│   ├── selected-domains.json
│   └── execution-history.json
├── artifacts/
│   └── <NNN>-<slug>/           # One folder per workflow run
│       ├── prompt-spec.md
│       ├── implementation-plan.md
│       ├── execution-report.md
│       ├── review-report.md
│       └── improvement-plan.md
└── logs/
```

## Domain memory files — organized by layer

Each `.claude/f67/memory/domains/<domain>/` groups knowledge by technical layer so agents load only the layer they work on:

| File | Contains |
|---|---|
| `overview.md` | Purpose, boundaries, key concepts, related domains |
| `business-logic.md` | Business rules (numbered, verbatim), invariants, domain patterns |
| `backend.md` | APIs/routes, services, events, database schema for this domain |
| `web.md` | Web app: routes/pages, components, state, data fetching (only if a web app exists) |
| `mobile.md` | Mobile app: screens, components, navigation, offline behavior (only if a mobile app exists) |
| `tests.md` | What is tested, where, how to run |
| `related-files.json` | File → role map, split by layer: `backend`, `web`, `mobile`, `shared` |
| `graph.json` | Domain-scoped nodes/edges |
| `history.md` | Dated one-line entries per completed task/workflow, newest first |
| `known-issues.md`, `decisions.md` | Debt and domain decisions |

A frontend-only or backend-only project simply omits the layers it doesn't have. Layer files replace the former `api.md`/`database.md`/`ui.md`/`architecture.md`/`dependencies.md`/`patterns.md` split — fewer files, one read per layer.

Rules:

- Files marked with `<!-- f67:curated -->` on the first line are human-curated. Agents may append below a `## Learned` heading but must never rewrite curated sections.
- Only create files that have content. An empty stub is worse than a missing file.
- Keep each file under ~150 lines; dense factual bullets, no prose.

## Memory freshness — continuous updates, not end-of-workflow

Memory must never lag the code. The update cadence:

1. **After every completed task** (implement/improve/execute): the executing agent itself appends the delta before returning — one dated line in each affected domain's `history.md`, new/moved/deleted paths in `related-files.json`, and any new business rule under `business-logic.md → ## Learned`. This costs a few lines and keeps memory current at all times.
2. **After every workflow**: the memory-evolver runs in *delta mode* — folds the artifact into feature records, graphs, and decisions. Cheap, incremental.
3. **On `/f67-sync`**: full reconciliation against git history.

An agent that finishes a task without writing its memory delta has not finished the task.

## Graph format

All graph files use one schema (`schemas/graph.schema.json` in the plugin):

```json
{
  "version": 1,
  "updated": "2026-07-04T12:00:00Z",
  "nodes": [{ "id": "domain:billing", "type": "domain", "label": "Billing" }],
  "edges": [{ "from": "domain:billing", "to": "feature:partial-refunds", "type": "contains" }]
}
```

Node types: `domain`, `feature`, `service`, `repository`, `component`, `api`, `test`, `database`, `event`, `file`.
Edge types: `contains`, `implements`, `uses`, `persists-to`, `calls`, `tested-by`, `emits`, `listens-to`, `related-to`.

Agents traverse graphs to find context instead of scanning the repository.

## The pipeline — routed by complexity

The domain detector's complexity rating selects the route. Speed is a feature: never run the full pipeline on work that doesn't need it.

| Complexity | Route |
|---|---|
| `trivial` / `small` | **Fast path** — `/f67-execute`: one executor agent does scoped context + implement + test + memory delta in a single dispatch |
| `medium` | Compact pipeline — spec and plan phases run with merged stages (see command skills) |
| `large` | Full pipeline — every stage, every artifact |

Full pipeline:

```
Detect domains → [Load memory ∥ Discover files] → Build context → Build spec
→ Plan+Decompose → Implement (one task) → Test → Review → Improve → Evolve memory
```

Hard rules:

1. Understanding always precedes code — but understanding is scoped to the request, never exhaustive.
2. The orchestrator (main session) never implements, reviews, or reads large codebases. It dispatches agents and maintains state.
3. Agents receive references (paths, node ids, artifact paths), not full file contents, wherever possible.
4. Each stage reads the artifact of the previous stage, never conversation history.
5. `/f67-implement` executes exactly one task per invocation.
6. **Parallel dispatch**: the orchestrator runs independent agents concurrently — memory-loader and discovery together (discovery starts from graphs; the digest's gaps trigger at most one follow-up), per-domain discovery agents in parallel during `/f67-init`, and test generation for independent completed tasks in parallel.

## Token discipline — binding on every agent

F67's value is speed. Every agent and artifact has a hard budget; exceeding it is a defect.

| Output | Hard cap |
|---|---|
| Domain detection JSON | 25 lines |
| Memory digest | 100 lines |
| Discovery report | 80 lines |
| context.md | 120 lines |
| prompt-spec.md | 120 lines |
| implementation-plan.md incl. task tree | 150 lines |
| execution-report.md **per task section** | 25 lines |
| review-report.md | 80 lines |
| improvement-plan.md | 60 lines |
| Any user-facing report from a command | 10 lines |

Rules that make the caps achievable:

- **No code in artifacts. Ever.** Reference `path:line`; describe changes in one line each. Diffs live in git, not in reports.
- **No repetition across artifacts** — later artifacts reference earlier ones by section, never restate them.
- **Digest, don't dump** — an agent that pastes a file it read into its output has failed.
- **Read scoped** — agents read the file sections graphs point to, not whole files, not whole directories.
- **User reports are conclusions** — what happened, what's next, where the artifact is. No process narration, no restating the plan.

## Model routing

Match model cost to task difficulty. Agent frontmatter carries a default; the orchestrator overrides per dispatch when complexity warrants.

| Work | Model |
|---|---|
| Detection, memory loading/evolution deltas | `haiku` |
| Discovery, context, spec, decomposition, testing, improvement | `sonnet` |
| Planning, implementation, review — and any dispatch for `large` complexity | `inherit` (the user's chosen top model) |

Simple task ⇒ fast model. Complex or correctness-critical ⇒ powerful model. Never the reverse.

## Artifact contracts

| Artifact | Producer | Consumers |
|---|---|---|
| `prompt-spec.md` | f67-prompt-builder agent | planner, decomposer, implementer, tester, reviewer |
| `implementation-plan.md` | f67-planner + f67-task-decomposer | implementer, tester |
| `execution-report.md` | f67-implementer (appended per task) | tester, reviewer, memory-evolver |
| `review-report.md` | f67-reviewer | improver, memory-evolver |
| `improvement-plan.md` | f67-improver | implementer, memory-evolver |

Artifact folders are numbered sequentially: `001-partial-refunds`, `002-fix-invoice-tax`, …

## State discipline

State files hold references only — paths, ids, task numbers, status enums. Never store code or long text in state. Example `current-plan.json`:

```json
{
  "artifact": ".claude/f67/artifacts/001-partial-refunds",
  "spec": "prompt-spec.md",
  "plan": "implementation-plan.md",
  "tasks": [
    { "id": "T1", "title": "Add refund amount to schema", "status": "done" },
    { "id": "T2", "title": "Refund service partial path", "status": "pending", "dependsOn": ["T1"] }
  ],
  "nextTask": "T2"
}
```

## config.yaml (runtime)

```yaml
version: 1
project:
  stack: []            # detected by /f67-init from the repository
  architecture: ""     # detected architecture style
  testFramework: ""    # detected test framework
  uiFramework: ""      # detected UI framework / styling approach
memory:
  autoEvolve: true     # run memory evolution after workflows
  protectCurated: true
skills:
  available: []        # resolved skills discovered/installed for this project, filled by /f67-init and /f67-sync
  requested: []        # skills F67 recommended but the user hasn't added yet
```

## Skill injection

F67 ships no stack-specific knowledge. It ships *rules* for discovering, injecting, and requesting skills (`${CLAUDE_PLUGIN_ROOT}/templates/skill-injection-rules.md`). Skills are resolved during context building, declared in the prompt-spec, and consumed by the planner, implementer, tester, and reviewer. Never load all available skills — only the mapped ones.

### Skill sources (resolution order)

1. **Project skills** — `.claude/f67/skills/*.md`: rules specific to this project (business domains, the project's stack conventions).
2. **Installed skills** — Claude Code skills and plugins already installed in the user's environment. Match by comparing skill names/descriptions against the request's technical areas.
3. **Requested skills** — nothing matched: F67 must not silently proceed. Recommend that the user install a matching skill from a plugin marketplace, or offer to generate a project skill draft into `.claude/f67/skills/` from the repo's own conventions and memory. Record the recommendation in `config.yaml → skills.requested`.

### Technical-area → skill category mapping

The domain detector tags each request with technical areas; the context builder maps areas to skill *categories* and searches the sources above for matches:

| Technical area | Skill categories to search for |
|---|---|
| frontend-ui | the project's frontend framework, component patterns |
| frontend-design | design system, UI/UX best practices |
| backend-api | the project's backend framework, API design |
| database | the project's ORM/database, data modeling, migrations |
| business-logic | the affected business domain, domain modeling (DDD), TDD |
| auth | authentication/authorization, security |
| infrastructure | deployment, CI/CD, containers, observability |
| testing | the project's test framework, testing strategy |

When no skill matches a category, the rules are derived from the project itself (configs, discovered patterns, memory, guidance files) per `skill-injection-rules.md`, and the gap becomes a skill request to the user. F67 never substitutes generic, non-project knowledge for a missing skill.

### Business-specific skills

F67 should actively suggest that users create project skills for their recurring business domains (e.g. a `billing` skill encoding invoice rounding rules, a `claims-processing` skill). Trigger this suggestion when: `/f67-init` completes, a domain accumulates 3+ features, or the memory evolver records repeated corrections in one domain. Project skills live in `.claude/f67/skills/<name>.md` and are auto-mapped by domain name.

### Methodology recommendations

- **TDD for business logic**: when a request's primary work is domain/business rules (requestType `feature` touching business-rules), the planner defaults to the `tdd` strategy and says so explicitly. Deviating requires a stated reason.
- **DDD or feature-sliced for greenfield**: when the project is new, or the request builds a capability with no existing implementation (no related features, thin domain memory), recommend structuring it with DDD (backend/domain-heavy) or feature-sliced architecture (frontend-heavy) before planning. `/f67-init` on a young repo and `/f67-brainstorm` must surface this recommendation; the planner records the choice as a decision record.
