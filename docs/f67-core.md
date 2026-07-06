# F67 core conventions

This document is the single source of truth for F67 runtime layout, workflow pipeline, and artifact contracts. Every F67 skill and agent must follow it.

## Runtime layout (inside the target repository)

All F67 project knowledge lives in the repository being worked on, never in the plugin:

```
.claude/f67/
├── config.yaml                 # Project-level F67 configuration
├── memory/
│   ├── index.json              # THE entry point: domain list, keywords, summaries,
│   │                           # layers, per-domain updatedAt, lastSyncCommit
│   ├── global/                 # architecture.md, coding-standards.md, security.md,
│   │                           # testing.md, ui.md, design-system.md, glossary.md, conventions.md
│   ├── domains/<domain>/       # One folder per business domain (see Domain memory files)
│   ├── features/<feature>/     # feature.md + graph.json per completed feature
│   ├── decisions/              # ADR-style decision records (NNNN-title.md)
│   └── sessions/               # Session summaries (YYYY-MM-DD-slug.md)
├── graphs/
│   ├── domain-graph.json       # Domain ↔ domain relationships
│   └── dependency-graph.json   # Coarse service/API/event edges across domains
│                               # (per-file mapping lives in each domain's related-files.json — no monolithic file graph)
├── state/
│   ├── current-session.json    # Active command, artifact path, stage
│   ├── current-spec.json       # Pointer to active prompt-spec artifact
│   ├── current-plan.json       # Task ids/status/nextTask ONLY — content lives in the plan doc
│   ├── changed-files.json      # path + change + task id, current workflow only
│   └── execution-history.json  # Completed workflows, lastSyncCommit
├── artifacts/
│   └── <NNN>-<slug>/           # One folder per workflow run
│       ├── prompt-spec.md
│       ├── implementation-plan.md
│       ├── execution-report.md
│       ├── review-report.md
│       └── improvement-plan.md
└── logs/
    └── metrics.jsonl           # One line per workflow: stages, dispatches, artifact sizes, duration
```

## memory/index.json — the entry point

Every F67 command starts by reading this one small file. It replaces the domain-detection agent dispatch entirely — the orchestrator classifies the request against it directly in-session.

```json
{
  "updated": "",
  "lastSyncCommit": "",
  "domains": {
    "billing": {
      "summary": "Invoicing, payments, refunds",
      "keywords": ["invoice", "refund", "payment", "tax"],
      "layers": ["backend", "web"],
      "features": 4,
      "updatedAt": ""
    }
  },
  "greenfieldHint": false
}
```

Maintained by the memory-evolver on every delta and by `/f67-sync`. If `index.json` is missing, memory predates this format — regenerate it from the domain folders (cheap) before proceeding.

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

## Memory freshness — enforced, not hoped for

Memory must never lag the code. Four mechanisms keep it current:

1. **After every completed task** (implement/improve/execute): the executing agent itself writes the delta before returning — dated line in each affected domain's `history.md`, path changes in `related-files.json`, new rules under `business-logic.md → ## Learned`, and the domain's `updatedAt` in `index.json`.
2. **After every workflow** (the review/improve stage or wherever the workflow ends): the orchestrator dispatches the memory-evolver in *delta mode* — folds the artifact into feature records, graphs, decisions, and the index. This is not optional and not deferred to `/f67-sync`.
3. **Staleness check at every command start**: after reading `index.json`, the orchestrator compares `lastSyncCommit` with `git rev-parse HEAD`. If the repo has moved significantly (work merged from other machines/teammates, or any domain's memory contradicts what discovery finds), it says so in one line and recommends `/f67-sync` — for `large` workflows it should run the sync delta before building context, because stale memory produces wrong specs.
4. **On `/f67-sync`**: full reconciliation against git history.

An agent that finishes a task without writing its memory delta has not finished the task. A workflow that ends without the evolver delta is not finished either.

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

Graphs stay coarse and global (domain relationships, cross-domain service/event edges). Per-file knowledge lives in each domain's layer-split `related-files.json` — that is the file graph, sharded by domain, cheap to keep fresh. Agents traverse index → domain graph → related-files instead of scanning the repository.

## The pipeline — routed by complexity

The **orchestrator itself** classifies each request by reading `memory/index.json` (domains, keywords, summaries) — no detection dispatch, no round-trip. It records the classification (domains, technical areas, complexity, greenfield) in `state/current-session.json` and routes. Speed is a feature: never run the full pipeline on work that doesn't need it. When classification is genuinely uncertain between two routes, take the heavier one — misclassifying down costs quality, misclassifying up only costs tokens.

| Complexity | Route |
|---|---|
| `trivial` / `small` | **Fast path** — `/f67-execute`: one executor agent does scoped context + implement + test + memory delta in a single dispatch |
| `medium` | Compact pipeline — spec and plan phases run with merged stages (see command skills) |
| `large` | Full pipeline — every stage, every artifact |

Full pipeline:

```
Classify (orchestrator, in-session) → [Load memory ∥ Discover files] → Build context
→ Build spec → Plan+Decompose → Implement (one task) → Test → Review → Improve → Evolve memory (delta)
```

Hard rules:

1. Understanding always precedes code — but understanding is scoped to the request, never exhaustive.
2. The orchestrator (main session) never implements, reviews, or reads large codebases. It dispatches agents and maintains state.
3. Agents receive references (paths, node ids, artifact paths), not full file contents, wherever possible.
4. Each stage reads the artifact of the previous stage, never conversation history.
5. `/f67-implement` executes exactly one task per invocation.
6. **Parallel dispatch**: the orchestrator runs independent agents concurrently — memory-loader and discovery together (discovery starts from graphs; the digest's gaps trigger at most one follow-up), per-domain discovery agents in parallel during `/f67-init`, and test generation for independent completed tasks in parallel.

## Output economy — purpose contracts, not line counts

There are no numeric caps. Instead, every artifact section has a stated consumer, and the producing agent is responsible for writing exactly what that consumer needs to act — nothing more, nothing less. An agent pads or truncates only if it has stopped thinking about its consumer. Scale output with the work: a two-file fix earns a short report; a twelve-task feature earns a longer one. Both are correct.

Two structural bans remain, because they caused real multi-thousand-line artifacts and have no consumer:

- **No code or diffs in artifacts. Ever.** Reference `path:line`; describe each change in prose. Diffs live in git.
- **No restating prior artifacts.** Later stages cite earlier ones by section.

And three working habits:

- **Digest, don't dump** — an agent that pastes a file it read into its output has failed its consumer.
- **Read scoped** — read the sections the index/graphs point to, not whole files or directories.
- **User reports are headlines.** The user sees one line per outcome — what was done, result, next step — like a changelog entry, not a summary. No process narration, no restating artifacts, no closing recaps. Details exist in the artifacts; expand only when the user explicitly asks for details or an explanation. A command whose chat output is longer than five short lines has failed this rule (exceptions: /f67-explain and /f67-brainstorm, whose product IS the text, and blocking questions that need the user's answer).

## Model policy — decided by the orchestrator at dispatch time

No agent has a fixed model. The orchestrator chooses per dispatch based on stakes, not on agent identity:

- **Default: the session's model.** Quality work — anything that writes memory, judges code, plans, implements, classifies, or builds specs — gets the most capable model available. Memory is the system's brain; polluting it to save tokens is the most expensive mistake F67 can make.
- **Downgrade only provably mechanical dispatches**: pure file-map/graph bookkeeping in `/f67-sync`, regenerating `index.json` from existing folders, formatting a session summary. If the dispatch requires judgment about code or business meaning, it is not mechanical.
- When in doubt, don't downgrade.

## Artifact contracts

| Artifact | Producer | Consumers |
|---|---|---|
| `prompt-spec.md` | f67-prompt-builder agent | planner, decomposer, implementer, tester, reviewer |
| `implementation-plan.md` | f67-planner + f67-task-decomposer | implementer, tester |
| `execution-report.md` | f67-implementer (appended per task) | tester, reviewer, memory-evolver |
| `review-report.md` | f67-reviewer | improver, memory-evolver |
| `improvement-plan.md` | f67-improver | implementer, memory-evolver |

Artifact folders are numbered sequentially: `001-partial-refunds`, `002-fix-invoice-tax`, …

## State discipline — one source of truth

Task *content* (descriptions, files, criteria, skills) lives only in `implementation-plan.md`. State files are pointers and status — duplicating content across plan and state caused real drift and 500+-line state files. Example `current-plan.json`, complete:

```json
{
  "artifact": ".claude/f67/artifacts/001-partial-refunds",
  "tasks": { "T1": "done", "T2": "pending", "T3": "pending" },
  "dependsOn": { "T2": ["T1"], "T3": ["T1"] },
  "parallelizable": [["T2", "T3"]],
  "nextTask": "T2"
}
```

`changed-files.json` is `{ "path": "...", "change": "added|modified|deleted", "task": "T1" }` entries for the current workflow only — no notes, no summaries. It resets when a new artifact folder is created (history lives in git and in domain memory).

## Metrics — measure, don't guess

At the end of every command, the orchestrator appends one line to `logs/metrics.jsonl`: `{ "ts": "", "command": "", "complexity": "", "dispatches": 0, "parallel": 0, "artifactBytes": 0, "outcome": "", "findingCategories": [] }` (`findingCategories` from review workflows). `/f67-memory audit` reports aggregates so optimization decisions are made on the team's real usage, not intuition.

## Learning from critique — recurring findings become conventions

Praise teaches memory what to repeat; repeated criticism must teach it what to prevent. The reviewer tags every finding with a category (e.g. `input-validation`, `error-handling`, `naming`, `authz`, `duplication`). When the memory evolver sees the same category across 3 recent workflows (from metrics + review reports), the finding is no longer a finding — it is a missing convention: the evolver proposes a one-line rule for `memory/global/` (or a project skill) to the user, and records acceptance as a decision. Prevention beats re-detection.

## Memory aging

Memory files must stay permanently cheap to load. The evolver compacts as it goes: `history.md` entries older than the last 10 workflows collapse to one line per feature (detail lives in the feature record); resolved items leave `known-issues.md`; superseded `## Learned` entries are folded into the curated sections' spirit by proposing the edit to the user (never silently). Growth without compaction is a defect.

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
  map: {}              # technicalArea -> resolved skill (project path or installed skill name),
                       # built by /f67-init, re-resolved by /f67-sync — per-request injection is a lookup
  requested: []        # skills F67 recommended but the user hasn't added yet
```

## Skill injection

F67 ships no stack-specific knowledge. It ships *rules* for discovering, injecting, and requesting skills (`${CLAUDE_PLUGIN_ROOT}/templates/skill-injection-rules.md`). Skills are resolved during context building, declared in the prompt-spec, and consumed by the planner, implementer, tester, and reviewer. Never load all available skills — only the mapped ones.

### Resolution is orchestrator-owned and cached

Only the orchestrator (main session) can see the user's installed skills and plugins — subagents cannot. So resolution happens once, in the orchestrator, and is cached:

1. **At `/f67-init`** (and re-resolved at `/f67-sync`): the orchestrator matches each technical area the project needs against project skills (`.claude/f67/skills/*.md`) first, then installed Claude Code skills/plugins, and writes the result to `config.yaml → skills.map`.
2. **Per request**: injection is a dictionary lookup — the orchestrator reads `skills.map` for the request's technical areas and passes the resolved names/paths to agents in the dispatch. No searching, no re-matching.
3. **Unmapped areas**: F67 must not silently proceed on priors. The gap is recorded in `skills.requested`, surfaced to the user (install from a marketplace, or let F67 generate an evidence-based project skill), and agents derive rules from project evidence per `skill-injection-rules.md` meanwhile.

### Technical-area → skill category mapping

The orchestrator's classification tags each request with technical areas; these are the `skills.map` keys resolved at init/sync:

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

When `skills.map` has no entry for a needed area, the rules are derived from the project itself (configs, discovered patterns, memory, guidance files) per `skill-injection-rules.md`, and the gap becomes a skill request to the user. F67 never substitutes generic, non-project knowledge for a missing skill.

### Business-specific skills

F67 should actively suggest that users create project skills for their recurring business domains (e.g. a `billing` skill encoding invoice rounding rules, a `claims-processing` skill). Trigger this suggestion when: `/f67-init` completes, a domain accumulates 3+ features, or the memory evolver records repeated corrections in one domain. Project skills live in `.claude/f67/skills/<name>.md` and are auto-mapped by domain name.

### Methodology recommendations

- **TDD for business logic**: when a request's primary work is domain/business rules (requestType `feature` touching business-rules), the planner defaults to the `tdd` strategy and says so explicitly. Deviating requires a stated reason.
- **DDD or feature-sliced for greenfield**: when the project is new, or the request builds a capability with no existing implementation (no related features, thin domain memory), recommend structuring it with DDD (backend/domain-heavy) or feature-sliced architecture (frontend-heavy) before planning. `/f67-init` on a young repo and `/f67-brainstorm` must surface this recommendation; the planner records the choice as a decision record.
