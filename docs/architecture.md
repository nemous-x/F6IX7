# F67 architecture

## Roles

The main Claude Code session is the orchestrator. It receives `/f67-*` commands, maintains lightweight state under `.claude/f67/state/`, dispatches agents, merges their summaries, and reports to the user. It never implements, reviews, or loads large contexts — agents return digests and artifact paths, not raw content.

## The thirteen agents

| # | Agent | Stage | Writes |
|---|---|---|---|
| 1 | f67-domain-detector | Detect domains | selected-domains.json |
| 2 | f67-memory-loader | Load DDM | memory digest (in-flight) |
| 3 | f67-discovery | Discover code | discovery report (in-flight) |
| 4 | f67-context-builder | Build context | context.md, active-context.json |
| 5 | f67-prompt-builder | Build spec | prompt-spec.md, current-spec.json |
| 6 | f67-planner | Plan | implementation-plan.md, current-plan.json |
| 7 | f67-task-decomposer | Decompose | task tree, current-plan.json |
| 8 | f67-implementer | Implement one task | code, execution-report.md, state |
| 9 | f67-tester | Test | tests, execution-report.md §Testing |
| 10 | f67-reviewer | Review | review-report.md |
| 11 | f67-improver | Improve | improvement-plan.md (+ applied fixes) |
| 12 | f67-memory-evolver | Evolve memory | DDM updates, graphs, sessions |
| 13 | f67-executor | Fast path (trivial/small) | code + memory delta + fast log, one dispatch |

Single responsibility is enforced by each agent's rules section: implementer executes exactly one task; tester never weakens assertions or patches implementation; reviewer never edits code; evolver never touches application code.

## Command → agent mapping

- `/f67-init` → discovery (×N, scoped) + memory-evolver, orchestrated in four phases with user confirmation of domain boundaries.
- `/f67-prompt` → detector → memory-loader → discovery → context-builder → prompt-builder.
- `/f67-plan` → planner → task-decomposer.
- `/f67-execute` → executor (fast path, one dispatch, self-scoped).
- `/f67-implement` → implementer (one task; parallel implementers for independent tasks).
- `/f67-test` → tester. `/f67-review` → reviewer. `/f67-improve` → improver.
- `/f67-discover`, `/f67-explain`, `/f67-brainstorm`, `/f67-memory` → read-only combinations of detector/memory-loader/discovery.
- `/f67-sync` → memory-evolver (+ scoped discovery for thin areas).
- `/f67-docs` → memory-loader + discovery; orchestrator writes docs (authoring style is a session-level concern).

## Why artifacts, not conversation

Every stage persists its output as a markdown artifact and consumes its predecessor's artifact. This makes workflows resumable across sessions, auditable, and deterministic — the same spec plus the same plan yields the same task tree regardless of chat history. State files carry only references (paths, task ids, status enums).

## Why graphs

Graphs (`domain-graph`, `file-graph`, `dependency-graph`) let agents find context by traversal instead of repo-wide scanning: domain → files → dependencies → tests in a few reads. Discovery falls back to targeted grep only for gaps, and full-repo scans are treated as a failure mode. `/f67-sync` keeps graphs honest; the discovery agent reports drift whenever the graph and the working tree disagree.

## Extension points

- **Skills** — F67 ships injection rules, not stack content (`templates/skill-injection-rules.md`). Projects add skills under `.claude/f67/skills/`; installed Claude Code skills are discovered and injected by category; missing categories become skill requests to the user.
- **New domains** — created by init or proposed by the domain detector (`newDomain`), confirmed by the user.
- **New strategies** — add to the planner's strategy enum with selection criteria.
- **Custom templates** — projects may shadow plugin templates by placing equivalents under `.claude/f67/templates/` (agents check project first, plugin second).
