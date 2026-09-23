# Diagnostics

Use read-only product tools before changing configuration or running an apply path.

## Fast path

1. `hermes plugins`: confirm `hermes-trove` is enabled and the selected context engine is `trove`.
2. Send one normal message if the session has not been bound since restart.
3. **Programmatic access**: `trove_status`/`trove_inspect`/`trove_doctor` are Python functions in `/home/ubuntu/hermes-trove/tools.py`, NOT CLI commands. They require a `TROVEEngine` instance passed as `engine=` kwarg via `importlib.util`. See SKILL.md for the full pattern.
4. If `TROVEEngine()` raises `database disk image is malformed`, follow the corruption recovery procedure in SKILL.md before attempting any diagnostics.
5. `trove_doctor`: run database, FTS, lifecycle, configuration, and context-pressure diagnostics (returns `overall: healthy` on success).

If optional slash commands are enabled, `/trove status` and `/trove doctor` expose the corresponding operator views. To enable: set `TROVE_ENABLE_SLASH_COMMAND=1` in the environment. Without it, the `/trove` slash commands are silently not registered.

## Safe mutation order

For cleanup, repair, source normalization, or rotate:

1. run the read-only preview;
2. inspect exact candidates and paths;
3. create/confirm a backup;
4. obtain user authorization for the specific apply operation;
5. run one bounded apply and verify integrity afterward.

Cleanup apply is separately feature-gated. Never infer permission to enable it from a diagnosis request.

## Common states

- Unbound status after restart: send a normal message, then check again.
- Database exists but stays empty: verify plugin enablement, `context.engine`, profile, database path, and ignore/stateless patterns.
- Weak exact recall: verify source rows exist, query construction/scope is correct, summary health is sound, and embedding coverage/provenance matches the requested mode.
- Conflicting summary and raw evidence: prefer the newer exact raw evidence and inspect lineage.
- Path B/context-engine schema log: expected on hosts where plugin-registry handlers do not receive active messages; context-engine schemas and dispatch remain the healthy route.
