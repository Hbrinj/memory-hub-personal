# memory-hub CLI — Deferred / Out-of-scope TODO

Items intentionally excluded from v1 (see `plans/memory-hub-cli-v1.md`). Pull from this list when scoping v2 or follow-up work.

## v2 commands
- [ ] `memory-hub search <query>` — grep across hub + local; later wraps Memsearch.
- [ ] `memory-hub index` — regenerate `.memsearch/milvus.db` for a hub.
- [ ] `memory-hub doctor` — diagnose pointer / hub / index health.
- [ ] `memory-hub unlink` — remove `.memory-hub` and (optionally) the local `memory/` scaffold.
- [ ] `memory-hub pin <sha>` — update the `pin` field in `.memory-hub`.

## Architecture / integrations
- [ ] Memsearch integration (vector index over hub + local).
- [ ] MCP integration so agents can query the hub directly.
- [ ] Hub push policy / governance docs (who pushes, how reviews work).

## Process / infra
- [ ] CI for the `memory-hub-cli` repo (build, lint, vitest on PRs).
- [ ] Release automation (changesets or similar).

## Explicitly NOT planned (design decisions, not TODO)
- Federation across hubs — strong isolation is the goal, do not add cross-hub backends.
- Automated promotion — `promote` stays an explicit command.
- Shared team-server backend — each hub is local-first.
- GUI — CLI-only.
