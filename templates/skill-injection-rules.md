# F67 skill injection rules

F67 does not ship stack-specific knowledge. This file defines how agents discover, inject, and request skills, plus stack-agnostic engineering baselines to fall back on when no skill is available. Project memory and project skills always override these baselines.

## Rule 1 — Discover before you inject

For each technical area tagged on the request, search in order and stop at the first match:

1. **Project skills** (`.claude/f67/skills/*.md`) — match by filename and first-line description against the area and affected domains.
2. **Installed skills** — the user's installed Claude Code skills and plugins. Match the area against skill names and descriptions (e.g. a request tagged `frontend-ui` in a React repo matches an installed React or frontend skill). Reference matched skills by name in the spec so downstream agents invoke them.
3. **No match** → apply Rule 2 and use the baselines below for this workflow.

Never inject skills for areas the request doesn't touch, and never load every available skill.

## Rule 2 — Request missing skills

When a needed category has no matching skill, F67 must surface a skill request to the user — during `/f67-init` (from the detected stack), or in the prompt-spec's skill section during workflows. The request must name: the category, why the project needs it (detected stack/requirement), and the two remedies:

- install a matching skill or plugin from a marketplace, or
- let F67 generate a project skill draft into `.claude/f67/skills/` from the repository's own conventions, memory, and guidance files.

Record open requests in `config.yaml → skills.requested`; the memory evolver clears entries once a matching skill appears.

## Rule 3 — Generate project skills from the project, not from priors

When drafting a project skill, derive every rule from evidence: linter configs, existing patterns discovery found, memory files, decision records. A generated skill with rules the repo doesn't actually follow is worse than none. Mark generated skills with `<!-- f67:generated -->` so users know to review them.

## Per-category engineering baselines (fallback only)

Used when no skill matched a category. General by design — never assume a specific framework.

### Frontend
Follow the framework conventions already in the repo. Components small and single-purpose; state as local as possible; every async boundary has loading/error/empty states; accessibility is non-negotiable (semantics, keyboard, focus, contrast, labels).

### Design / UI
Use the project's design tokens and component inventory before inventing anything. One primary action per view. All interactive states designed (hover, focus, disabled, loading, error). Consistency with existing screens beats novelty.

### Backend / API
Match the existing API style exactly (protocol, versioning, error shape). Validate at every input boundary; map internal entities to response DTOs; correct status/error semantics; idempotency for state- or money-critical operations; pagination on collections.

### Database
Schema changes ship with migrations; multi-write operations are transactional; watch for N+1 access patterns; constraints and defaults live in the schema, not application patch-up code.

### Business logic
Keep domain rules in the domain layer, expressed in the project's ubiquitous language (glossary + domain memory). Default to TDD (see methodology rules in f67-core). Every business rule implemented must also land in the domain's `business-rules.md`.

### Auth / security
Never roll custom crypto or session mechanisms. Default-deny on new routes. Authorization enforced server-side, not by UI hiding. No secrets in code, logs, or fixtures. Enumeration-safe account flows. Auth changes require a decision record.

### Infrastructure
Config via environment with validation at boot; secrets via the project's secret mechanism; every deployable change has a rollback path; new services get structured logs and at least one alert-worthy metric.

### Testing
Follow `memory/global/testing.md`. Test behavior through public interfaces; one behavior per test, named as a statement; use the project's factories; cover boundaries, null/empty, concurrency, authorization, and failure paths that matter to the domain's rules.

### Greenfield structure
When building something with no existing implementation, recommend DDD-style structure for domain/backend-heavy work or feature-sliced structure for frontend-heavy work, record the choice as a decision, and keep dependency direction strict from day one.
