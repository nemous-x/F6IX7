---
name: f67-discover
description: >
  Performs intelligent F67 discovery — detects affected domains, collects relevant files
  via graph traversal, and updates the active context without planning or coding. Trigger
  with "/f67-discover [topic]", "discover the f67 context for", or "which files and
  domains relate to".
---

# /f67-discover — Intelligent discovery

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`. Requires `.claude/f67/` (else point to `/f67-init`).

## Pipeline

1. Dispatch `f67-domain-detector` with the topic. Save to `state/selected-domains.json`.
2. Dispatch `f67-memory-loader` for the detected domains.
3. Dispatch `f67-discovery` with the digest.
4. Update `state/active-context.json` with the discovery report path and file list. If an artifact folder is active, save the report there as `discovery.md`; otherwise save under `.claude/f67/artifacts/adhoc-discovery/`.

## Report

Present to the user: affected domains, entry points, dependency flow, relevant tests, and any graph drift found (suggest `/f67-sync` if drift is significant). No plans, no code changes.
