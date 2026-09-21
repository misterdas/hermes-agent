---
name: hermes-lcm
description: Use, configure, diagnose, and retrieve exact evidence with the Hermes-LCM lossless context plugin.
---

# Hermes-LCM

Use this skill when a task concerns Hermes-LCM setup, operation, compaction, diagnostics, session behavior, or recall from compacted and cross-conversation history.

Start here:

1. Confirm that the `hermes-lcm` plugin is enabled and `context.engine` is `lcm`.
2. For exact historical claims, use the recall workflow instead of trusting a compacted summary.
3. Use `lcm_status`, `lcm_inspect`, and `lcm_doctor` before changing configuration or attempting repair.
4. Treat slash-command apply paths as mutations: preview first, keep backups, and require the user's authorization.
5. Load the relevant reference rather than guessing arguments or lifecycle semantics.

Reference map:

- Configuration and activation: `references/configuration.md`
- Architecture and data ownership: `references/architecture.md`
- Diagnostics and safe operator workflow: `references/diagnostics.md`
- Recall tools and routing: `references/recall-tools.md`
- `/new`, session continuity, and `/lcm rotate`: `references/session-lifecycle.md`
- Canonical runtime recall policy: `references/recall-policy.md`

Working rules:

- Raw stored messages are authoritative; summaries are bounded recall cues.
- Prefer newer source-backed evidence when it conflicts with an older summary.
- Start with the narrowest useful scope and expand only when exact detail is needed.
- Do not infer exact commands, paths, timestamps, values, counts, or causal chains from summaries alone.
- Keep current-session, cross-conversation, and Hermes history outside `lcm.db` distinct.
- Do not treat open-cardinality results as complete without product-verifiable enumeration or coverage.
- Use `lcm_compile_evidence` when a historical answer needs several named facets, exact operands, conflict handling, or latest-state selection; treat its semantic proposal as untrusted until the product returns validated evidence.
- Keep default-off assertion, query-view, adaptive-retrieval, and destructive operator paths default-off unless the user explicitly asks to enable them.

## Critical Bug Workarounds (verified in production)

These bugs have been identified and fixed in the upstream repo but may persist in older versions:

- **#614 — L3 truncation exceeds source_tokens**: `summarize_with_escalation()` can return L3 deterministic truncation larger than the source chunk. Fixed: L3 `max_tokens` is now bounded by `min(l3_truncate_tokens, source_tokens - 1)`.
- **#599/#606 — Storage duplicate re-ingest**: Without `identity_hash`, replayed/compacted messages re-INSERT as duplicates (65% duplicate rows in production, 608K→1.31M session doubling). Fixed: `identity_hash` column + `INSERT OR IGNORE` on SHA-256 of session+role+content+timestamp. Auto-migrates existing databases.
- **#614 circuit breaker**: Rejected results (valid LLM output too large) were recorded as circuit-breaker failures, opening circuits prematurely. Fixed: `record_failure` only fires on actual LLM errors (exception/None), not size-rejected results.

### Upstream open issues (monitor before upgrading)

The `hermes-lcm` repo at `stephenschoettler/hermes-lcm` has ~30 open issues including:
- `#601` — Background rollup corruption (`btreeInitPage` error 11, deleted WAL handles)
- `#605` — SQLite store lazy reconnect + `__deepcopy__` missing
- `#589` — APFS concurrency corruption, lock timeouts, FTS self-healing
- `#588` — SQLite permission helpers drop POSIX locks
- `#607` — Session-end lock waits not bounded by wall clock
- `#608` — Unsatisfiable compaction thresholds clear fresh tail
- `#612` — Private key redaction via regex instead of linear scanner
- `#613` — Per-turn whole-transcript re-ingest on mutated tails

Before upgrading, check `git log` for fixes to issues affecting your setup. See the repo's CHANGELOG.md for version-by-version details.

## Version Compatibility

Hermes v0.21.3+ recommended. Earlier versions may have schema incompatibilities with the LCM plugin's auto-migrations.

## References
