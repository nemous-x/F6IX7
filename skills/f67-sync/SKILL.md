---
name: f67-sync
description: >
  Synchronizes F67 Domain Driven Memory with repository changes — updates graphs, file
  associations, and relationships, and folds completed workflow artifacts into memory.
  Trigger with "/f67-sync", "sync f67 memory", or "update f67 after these changes".
---

# /f67-sync — Synchronize memory with reality

Act as the F67 orchestrator. Read `${CLAUDE_PLUGIN_ROOT}/docs/f67-core.md`.

## Change detection

1. Determine what changed since the last sync: `git diff --name-status` against `lastSyncCommit` from `memory/index.json` (fall back to comparing each domain's related-files.json against the working tree).
2. Classify changes: files added/moved/deleted, domains affected (via index keywords and each domain's related-files.json), completed workflow artifacts not yet folded into memory.

## Pipeline

1. Dispatch `f67-memory-evolver` with the change classification. It:
   - Updates each affected domain's `related-files.json` and the domain/dependency graphs. The pure bookkeeping part may run on a fast model; anything requiring judgment about code meaning gets the session model.
   - Folds unprocessed workflow artifacts into domain memory, feature records, decisions, and session summaries.
   - Applies pending memory corrections from `context.md` files.
2. For substantial new code in a domain with thin memory, dispatch `f67-discovery` scoped to that area first so the evolver has facts to record.
3. Record the synced commit in `memory/index.json → lastSyncCommit` and `state/execution-history.json`; refresh every touched domain's `updatedAt`.
4. Append the metrics line to `logs/metrics.jsonl`.

## Report

Conclusions only: domains updated, drift corrected, artifacts folded, and any domain needing `/f67-memory rebuild`.

Sync is additive and incremental — it never rewrites curated memory and is safe to run any time.
