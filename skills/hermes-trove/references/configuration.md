# Configuration and activation

Hermes-TROVE is a general Hermes plugin and a context engine. Both identities must be active:

```yaml
plugins:
  enabled:
    - hermes-trove

context:
  engine: trove
```

Restart Hermes after changing plugin or context-engine configuration. Verify with `hermes plugins`, then use `trove_status` after a normal message has bound the session.

## Installation

An existing checkout can install profile-aware plugin and skill links:

```bash
./scripts/install.sh
HERMES_PROFILE=myprofile ./scripts/install.sh
```

The installer exposes both:

- `plugins/hermes-trove` for plugin loading;
- `skills/hermes-trove` for normal skill discovery.

It refuses conflicting paths rather than overwriting an existing install.

## High-impact controls

Use `docs/operator-guide.md` as the complete current source. Start with:

- `TROVE_CONTEXT_THRESHOLD`: when normal context pressure triggers compaction;
- `TROVE_FRESH_TAIL_COUNT`: newest messages kept raw;
- `TROVE_LEAF_CHUNK_TOKENS`: maximum raw material per leaf compaction group;
- `TROVE_DATABASE_PATH`: profile-local SQLite path when the default is unsuitable;
- `TROVE_IGNORE_SESSION_PATTERNS` and `TROVE_STATELESS_SESSION_PATTERNS`: storage ownership boundaries;
- summary/embedding provider settings only after confirming credentials, cost, and data handling.

Optional slash commands are disabled by default (`TROVE_ENABLE_SLASH_COMMAND=false`, mapped to `config.slash_commands_enabled`). To enable the `/trove` operator surface, set `TROVE_ENABLE_SLASH_COMMAND=1` in `~/.hermes/.env` (or set `slash_commands_enabled: true` in `config.yaml` if supported). Destructive cleanup apply is separately guarded. Do not enable mutation surfaces merely to diagnose a problem.

Change one tuning variable at a time, then re-check `trove_status`, context pressure, summary health, latency, and actual answer quality.
