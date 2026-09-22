# Session lifecycle and rotate

Hermes `/new` starts a new host session. Hermes-TROVE binds that session to its own lifecycle row and may carry eligible higher-depth summaries into the new current-session context. Source eligibility and exact expansion still come from descendant raw messages; carried summaries do not rewrite source ownership.

Do not promise that `/new` deletes historical TROVE data. Earlier rows remain in `trove.db` unless an explicitly authorized cleanup removes them, and they remain available through bounded cross-session recall.

## `/trove rotate`

`/trove rotate` is different from `/new`. It requires the slash-command surface to be enabled: set `TROVE_ENABLE_SLASH_COMMAND=1`. Without it, use the context-engine tool layer (`trove_rotate` / `trove_rotate_apply`) instead.

- it keeps the current `session_id` and `conversation_id`;
- preview is read-only;
- apply creates/updates the rolling rotate backup first;
- it preserves the configured fresh tail;
- it advances the lifecycle frontier past older raw messages so bootstrap does not replay them into active context;
- it does not delete raw source rows or call a summarization model.

Run normal compaction before rotate when older material must be represented in summary nodes. Even without a summary, pre-tail raw rows remain recoverable through `trove_load_session` and `trove_expand`.

Rotate refuses ignored or stateless sessions. Repeating an already-satisfied rotate reports a no-op and preserves the previous known-good rolling backup.

Use a separate session when the user wants a new active conversational boundary. Use rotate when the problem is active transcript/frontier size without changing identity.
