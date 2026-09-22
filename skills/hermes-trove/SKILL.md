---
name: hermes-trove
description: Use, configure, diagnose, and retrieve exact evidence with the Hermes-TROVE lossless context plugin.
---

# Hermes-TROVE

Use this skill when a task concerns Hermes-TROVE setup, operation, compaction, diagnostics, session behavior, or recall from compacted and cross-conversation history.

Start here:

1. Confirm that the `hermes-trove` plugin is enabled and `context.engine` is `trove`.
2. If you intend to use `/trove` slash commands, verify they are enabled: set `TROVE_ENABLE_SLASH_COMMAND=1` (e.g., in `~/.hermes/.env`). This maps to `config.slash_commands_enabled` via `TROVEConfig.from_env()`. Without it, `/trove` commands are silently not registered — `trove_status`, `trove_inspect`, and `trove_doctor` still work via the context-engine tool layer.
3. For exact historical claims, use the recall workflow instead of trusting a compacted summary.
4. Use `trove_status`, `trove_inspect`, and `trove_doctor` before changing configuration or attempting repair.
5. Treat slash-command apply paths as mutations: preview first, keep backups, and require the user's authorization.
6. Load the relevant reference rather than guessing arguments or lifecycle semantics.

Reference map:

- Configuration and activation: `references/configuration.md`
- Architecture and data ownership: `references/architecture.md`
- Diagnostics and safe operator workflow: `references/diagnostics.md`
- Recall tools and routing: `references/recall-tools.md`
- `/new`, session continuity, and `/trove rotate`: `references/session-lifecycle.md`
- Canonical runtime recall policy: `references/recall-policy.md`

Working rules:

- Raw stored messages are authoritative; summaries are bounded recall cues.
- Prefer newer source-backed evidence when it conflicts with an older summary.
- Start with the narrowest useful scope and expand only when exact detail is needed.
- Do not infer exact commands, paths, timestamps, values, counts, or causal chains from summaries alone.
- Keep current-session, cross-conversation, and Hermes history outside `trove.db` distinct.
- Do not treat open-cardinality results as complete without product-verifiable enumeration or coverage.
- Use `trove_compile_evidence` when a historical answer needs several named facets, exact operands, conflict handling, or latest-state selection; treat its semantic proposal as untrusted until the product returns validated evidence.
- Keep default-off assertion, query-view, adaptive-retrieval, and destructive operator paths default-off unless the user explicitly asks to enable them.

## Critical Bug Workarounds (verified in production)

These bugs have been identified and fixed in the upstream repo but may persist in older versions:

- **#614 — L3 truncation exceeds source_tokens**: `summarize_with_escalation()` can return L3 deterministic truncation larger than the source chunk. Fixed: L3 `max_tokens` is now bounded by `min(l3_truncate_tokens, source_tokens - 1)`.
- **#599/#606 — Storage duplicate re-ingest**: Without `identity_hash`, replayed/compacted messages re-INSERT as duplicates (65% duplicate rows in production, 608K→1.31M session doubling). Fixed: `identity_hash` column + `INSERT OR IGNORE` on SHA-256 of session+role+content+timestamp. Auto-migrates existing databases.
- **#614 circuit breaker**: Rejected results (valid LLM output too large) were recorded as circuit-breaker failures, opening circuits prematurely. Fixed: `record_failure` only fires on actual LLM errors (exception/None), not size-rejected results.

### CI Test Environment Pitfalls

- **SQLite directory ownership check (`sqlite_util.py`)**: `_open_private_sqlite_directory` verifies `st_uid == os.getuid()` (not `st_mode & 0o022`). A shell umask of `0002` creates group-writable `0o775` dirs, which the old mode-bit check falsely rejected. Always check directory OWNER, not mode bits.
- **Low-FD pytest**: CI runs `ulimit -n 1024; python -m pytest tests/ -q`. If tests fail under low FD, check `sqlite_util.py` ownership logic and temp-directory permissions — `tmp_path` + `Path.mkdir()` with umask `0002` triggers the old `0o022` check.

### Upstream open issues (monitor before upgrading)

The `hermes-trove` repo at `misterdas/hermes-trove` has ~30 open issues including:
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

Hermes v0.21.3+ recommended. Earlier versions may have schema incompatibilities with the TROVE plugin's auto-migrations.

## Release Procedure

Releasing a new version follows this sequence. Each step is verified before proceeding:

1. **Version bump**: Update `plugin.yaml` `version:` and all hardcoded version strings in tests (`test_trove_engine.py`, `test_trove_command.py`, `test_packaging_install.py`, `test_release_workflow.py`), `README.md`, `docs/operator-guide.md`, `CHANGELOG.md`, and `.github/ISSUE_TEMPLATE/bug_report.yml`. Use `grep -rln` to find all occurrences first.
2. **CHANGELOG restructure**: Move `## Unreleased` to become `## v{VERSION} - {DATE}`, then insert a fresh `## Unreleased` section at the top for future changes.
3. **Release notes**: Create `.github/release-notes/v{VERSION}.md` from the template. The release workflow at `.github/workflows/release.yml` requires this file to exist and start with `# hermes-trove v{VERSION}\n`.
4. **Commit and tag**: `git add -A && git commit -m "Release v{VERSION}" && git tag -a v{VERSION} -m "hermes-trove v{VERSION} release"`.
5. **Push**: `git pull --rebase origin main` first (remote may have new commits), then `git push origin main && git push origin v{VERSION}`.
6. **Verify**: Run `pytest tests/test_release_workflow.py tests/test_trove_command.py -q` to confirm test consistency.

**Pitfalls**:
- The tag-driven CI marks tags containing `-` as prereleases and non-`-` tags as `latest`. A stable release must use a tag without `-`.
- `git reset --hard HEAD~1` destroys uncommitted work — commit before resetting.
- `.github/release-notes/v{VERSION}.md` must exist before pushing the tag, or the release workflow rejects it.

## References
