# Agent configuration profiles

Copy-paste starting points for common agent shapes. Every profile below is
**additive to a stock install** — with none of these variables set, TROVE runs
exactly as before. Mix and match: the feature families are independent.

For the full variable table see
[Operator guide → Configuration](operator-guide.md#configuration); for what
each feature does and why, see [Feature overview](features-overview.md).

## Where configuration lives

1. **Environment variables (`TROVE_*`) are the primary surface** and always win.
   Set them in the environment that launches Hermes.
2. **`~/.hermes/config.yaml`** participates in three narrow, deliberate ways:
   - `plugins.enabled: [hermes-trove]` + `context.engine: trove` activate the
     plugin (see [Operator guide → Activate](operator-guide.md#activate));
   - `trove.context_threshold` is the one TROVE key supported in YAML (used only
     when `TROVE_CONTEXT_THRESHOLD` is not set; other keys under `trove:` are
     ignored and reported by `/trove doctor`);
   - when neither is set, TROVE inherits the Hermes global
     `compression.threshold`.
3. **Summarization inherits Hermes auxiliary routing.** Rollup builds and
   compaction summaries go through the auxiliary model unless you override
   `TROVE_SUMMARY_MODEL` — so a fully-local Hermes (local auxiliary model) makes
   every TROVE feature below local too, including temporal rollups.
4. Secrets stay in the environment: the only one TROVE ever reads is
   `VOYAGE_API_KEY`, and only when `TROVE_EMBEDDING_PROVIDER=voyage`.

Check what actually took effect at runtime with `/trove status` (reports config
sources) and `/trove doctor` (flags ignored YAML keys and misconfiguration).

## Profile: default chat assistant

Nothing to configure. Enable the plugin and stop:

```yaml
# ~/.hermes/config.yaml
plugins:
  enabled:
    - hermes-trove
context:
  engine: trove
```

You get bounded active context, the summary DAG, lossless recovery, and the
full `trove_*` tool set at their tested defaults.

## Profile: heavy tool-use coding agent

For agents that run builds, tests, linters, and searches all day. The goal is
to stop giant tool outputs from monopolizing the prompt while keeping every
byte recoverable.

```bash
# Externalize oversized payloads out of trove.db into recoverable files
export TROVE_LARGE_OUTPUT_EXTERNALIZATION_ENABLED=true

# Replace token-heavy tool results in the provider-visible prompt with refs
export TROVE_LARGE_OUTPUT_ACTIVE_REPLAY_STUBBING_ENABLED=true
# Default stub threshold is 25000 tokens; lower it for chattier tools
# export TROVE_LARGE_OUTPUT_ACTIVE_REPLAY_STUB_THRESHOLD_TOKENS=25000

# Cap the protected fresh tail by tokens (0 = off). Prevents one giant recent
# tool result from pinning the whole budget; newest message and complete
# assistant/tool groups are always retained.
export TROVE_FRESH_TAIL_MAX_TOKENS=24000

# Optional: at threshold, drain the whole raw backlog in one bounded sweep
# (fewer, larger compactions — useful after long unattended runs)
export TROVE_THRESHOLD_FULL_SWEEP_ENABLED=true
```

Recovery stays first-class: `trove_describe`/`trove_expand` read externalized
refs, and `trove_grep(content_scope='both')` searches externalized payloads when
the agent needs "which run printed that error?".

## Profile: long-horizon personal / companion agent

For agents that live for weeks and get asked "what did we do last Tuesday?"
and "have we talked about this before?".

```bash
# Time-indexed memory: day/week/month rollups + the trove_recent tool
export TROVE_TEMPORAL_ROLLUPS_ENABLED=true

# Meaning-based recall: semantic + hybrid trove_grep over summaries
export TROVE_EMBEDDINGS_ENABLED=true
export TROVE_EMBEDDING_PROVIDER=voyage          # or: fastembed / ollama (below)
export TROVE_EMBEDDING_MODEL=voyage-4-lite
export VOYAGE_API_KEY=...                     # free key from dash.voyageai.com

# The /trove operator commands used below are an opt-in surface
export TROVE_ENABLE_SLASH_COMMAND=1
```

Then, once:

```
/trove embed warmup            # resolve + dimension-lock the profile
/trove embed backfill          # dry-run: shows cost/coverage estimate
/trove embed backfill --apply  # embed existing summaries
```

`trove_recent` works immediately even before rollups are built (transparent
leaf-summary fallback), and `trove_grep` keeps `mode='full_text'` as the
byte-compatible default — semantic is per-call opt-in.

## Profile: fully local / air-gapped

No bytes leave the machine. Pair with a local Hermes auxiliary model so
summarization is local too.

```bash
export TROVE_TEMPORAL_ROLLUPS_ENABLED=true

export TROVE_EMBEDDINGS_ENABLED=true
export TROVE_EMBEDDING_PROVIDER=fastembed       # in-process ONNX, CPU-friendly
# Default model downloads ~90–130 MB once at warmup (bge-small / MiniLM class)

# Or, if you already run Ollama:
# export TROVE_EMBEDDING_PROVIDER=ollama
# export TROVE_EMBEDDING_MODEL=nomic-embed-text
# export TROVE_OLLAMA_BASE_URL=http://127.0.0.1:11434
```

Optional: installing `numpy` accelerates KNN on large vector sets; without it
TROVE uses a dependency-free bounded scan that stays correct, just smaller-scale
(see [Embeddings setup](embeddings-setup.md)).

## Profile: cost-guarded cloud embeddings

Voyage's free tier (200M tokens on the voyage-4 family) is far more than this
workload typically needs — TROVE embeds bounded summaries, not raw transcripts.
To keep hard boundaries anyway:

```bash
export TROVE_EMBEDDINGS_ENABLED=true
export TROVE_EMBEDDING_PROVIDER=voyage
export TROVE_EMBEDDING_MODEL=voyage-4-lite      # cheapest tier, $0.02/M after free
export VOYAGE_API_KEY=...

# Interactive queries: hard per-call wall-clock budget (default 3s)
export TROVE_EMBEDDING_QUERY_TIMEOUT_S=3

# Bulk backfill: separate per-request deadline (default 120s) — bulk work
# never inherits the interactive query budget
export TROVE_EMBEDDING_BACKFILL_TIMEOUT_S=120
```

Always dry-run `/trove embed backfill` first: it reports document counts and
token estimates with the same eligibility rules apply-mode uses, so the
estimate is the bill.

## Verifying a profile

```bash
hermes plugins        # plugin + engine loaded
```

The `/trove` operator commands below require the opt-in command surface:

```bash
export TROVE_ENABLE_SLASH_COMMAND=1
```

(Without it, the `trove_status` / `trove_doctor` *agent tools* still cover the
read-only checks — ask the agent to call them.) Then in a session:

```
/trove status           # effective config + sources, context pressure
/trove doctor           # DB health, ignored YAML keys, misconfig warnings
/trove rollups          # temporal rollup readiness (when enabled)
/trove embed backfill   # dry-run: embedding coverage + cost estimate (when enabled)
```
