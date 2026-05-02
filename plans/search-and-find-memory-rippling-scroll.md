# Plan: `memory-hub` CLI for Domain-Isolated Multi-Repo Memory

## Context
Goal: a memory architecture where multiple repositories belonging to the same **team / business unit** share knowledge, while different teams' memories stay strictly separated. Domain ≠ filesystem location — a team might have repos scattered anywhere on disk, and ownership can change without moving directories.

The architecture is **per-team memory hub repos** (each an independent git repo), with each consuming repo declaring its hub via a `.memory-hub` JSON pointer. Strong isolation, no cross-domain backend.

Doing this by hand — scaffolding wiki layouts, writing pointer files, deciding hub-vs-local, committing to the right repo — is fiddly and easy to get wrong. So this plan builds a **CLI tool (`memory-hub`)** that instantiates and maintains these hubs and the repos that consume them.

## Architecture (target end state)

```
~/development/
├── repo-mem/                         # FIRST HUB — the "personal" team
│   ├── AGENTS.md                     # schema (Karpathy wiki conventions)
│   ├── wiki/                         # entities, concepts, sources, synthesis
│   ├── .memsearch/milvus.db          # OPTIONAL local index, gitignored
│   └── .git/                         # its own repo
│
├── team-payments-memory/             # second HUB — payments team
│   └── … (same scaffold)
│
├── payments-api/                     # consuming repo
│   ├── .memory-hub                   # JSON pointer
│   ├── memory/                       # repo-LOCAL memory (private)
│   └── CLAUDE.md                     # two-tier read/write protocol
│
└── memory-hub-cli/                   # NEW — the CLI tool's own repo
    ├── src/
    ├── bin/
    └── package.json
```

`.memory-hub` format (unchanged from prior decision):
```json
{
  "path": "../team-payments-memory",
  "url": "git@github.com:org/team-payments-memory.git",
  "pin": "a1b2c3d",
  "branch": "main"
}
```

## CLI design

**Binary**: `memory-hub` (also aliased as `mh` for ergonomics)
**Runtime**: Node (TypeScript), distributed via npm — invocable as `npx memory-hub <cmd>` or installed globally with `npm i -g memory-hub-cli`.
**Source repo**: standalone `memory-hub-cli` repo, separate from any hub.

### v1 commands

| Command | Behavior |
|---|---|
| `memory-hub init [--team <name>] [--path <dir>]` | Scaffold a new hub: create `wiki/{index.md,log.md,entities/,concepts/,sources/,synthesis/}/`, `raw/`, `AGENTS.md`, `README.md`, `.gitignore` (excludes `.memsearch/`). Run `git init` and make the scaffold commit. Default `path` = `~/development/team-<name>-memory`. Special case: `--team personal` produces `repo-mem` and notes the naming exception in the README. |
| `memory-hub link --hub <path-or-url>` | In the current repo: write `.memory-hub` JSON, scaffold a local `memory/` directory with a minimal wiki layout, and write a `CLAUDE.md` containing the two-tier read/write protocol the agent should follow. Refuse to overwrite without `--force`. |
| `memory-hub status` | In the current repo: resolve `.memory-hub`, print the hub path/URL, hub HEAD SHA, ahead/behind vs `origin`, count of entries in hub vs local `memory/`, and any uncommitted changes in either tier. |
| `memory-hub add <kind> <slug> [--scope hub\|local] [--title "..."]` | Append a new wiki entry. `<kind>` ∈ `entity \| concept \| source \| synthesis`. Creates the markdown file with frontmatter (name, description, type, created date) and appends a line to `wiki/log.md` (`## [YYYY-MM-DD] add | <title>`). Default `--scope local`; promotion is a separate command. Opens `$EDITOR` after creation unless `--no-edit`. |
| `memory-hub promote <path>` | Move a file from the current repo's `memory/` to the linked hub's `wiki/` (preserving subpath), update both index.md files and the hub's log.md, then `git add` + commit in **both** repos with linked messages (`promote: <slug> from <repo>` in the hub, `promote: <slug> -> hub` in the consuming repo). Requires clean trees in both. |
| `memory-hub sync` | In the linked hub: `git fetch && git pull --ff-only`. Warn loudly if local hub commits are unpushed or if the working tree is dirty. Never auto-push. |

### Deferred to v2 (documented but not built)
- `memory-hub search <query>` — grep across hub + local; later wraps Memsearch.
- `memory-hub index` — regenerate `.memsearch/milvus.db` for a hub.
- `memory-hub unlink`, `memory-hub doctor`, `memory-hub pin <sha>`.

### Tech stack
- **Language**: TypeScript, compiled to `dist/` via `tsc`. Node ≥ 20.
- **CLI framework**: `commander` (small, well-known, good `--help` output).
- **Git**: shell out to `git` via `node:child_process` (avoid `simple-git` to stay dependency-light).
- **Filesystem**: `node:fs/promises`.
- **Templates**: inline string templates in `src/templates/` for `AGENTS.md`, `CLAUDE.md`, the wiki scaffold, and `.gitignore`. No template engine.
- **Tests**: `vitest`. Each command tested against a temp-dir fixture using real `git init`.
- **Lint/format**: `biome` (single tool, zero-config).

### Repo layout for `memory-hub-cli`
```
memory-hub-cli/
├── package.json              # bin: { "memory-hub": "./bin/memory-hub.js", "mh": "./bin/memory-hub.js" }
├── tsconfig.json
├── biome.json
├── bin/
│   └── memory-hub.js         # thin shim: require('../dist/cli.js')
├── src/
│   ├── cli.ts                # commander setup, dispatches to commands/
│   ├── commands/
│   │   ├── init.ts
│   │   ├── link.ts
│   │   ├── status.ts
│   │   ├── add.ts
│   │   ├── promote.ts
│   │   └── sync.ts
│   ├── lib/
│   │   ├── hub.ts            # resolve .memory-hub, find hub root
│   │   ├── git.ts            # thin wrapper over child_process git
│   │   ├── wiki.ts           # frontmatter, log append, index update
│   │   └── paths.ts          # default locations, slug validation
│   └── templates/
│       ├── AGENTS.md.ts
│       ├── CLAUDE.md.ts
│       ├── README.md.ts
│       ├── gitignore.ts
│       └── wiki/             # index.md, log.md, kind-specific stubs
└── tests/
    └── commands/*.test.ts
```

## Steps

### Phase 1 — Bootstrap the CLI repo
1. Create `~/development/memory-hub-cli/`, `git init`, scaffold `package.json` (Node 20, type=module, `bin` entries for `memory-hub` and `mh`), `tsconfig.json`, `biome.json`, `.gitignore` (`dist/`, `node_modules/`).
2. Add deps: `commander`. Dev deps: `typescript`, `vitest`, `@biomejs/biome`, `@types/node`.
3. Create `bin/memory-hub.js` shim and `src/cli.ts` skeleton with `commander` registering all six v1 commands as no-op stubs that print "not implemented".
4. Wire `npm run build` (tsc), `npm test` (vitest), `npm run lint` (biome).
5. Commit: `chore: scaffold memory-hub-cli`.

### Phase 2 — Implement `init` and `link` (the bootstrap path)
1. Build `src/templates/` for `AGENTS.md`, `README.md`, `.gitignore`, `CLAUDE.md`, and the wiki scaffold (`index.md`, `log.md`, plus empty `entities/`, `concepts/`, `sources/`, `synthesis/` directories with `.gitkeep`).
2. Implement `init` (creates hub, runs `git init`, makes scaffold commit). Handle the `--team personal` special case.
3. Implement `link` (writes `.memory-hub`, scaffolds local `memory/`, writes `CLAUDE.md`).
4. Tests: temp dir, run `init`, assert file tree + commit; then `link` from a sibling temp repo, assert pointer + local scaffold.
5. **Dogfood**: from `~/development/`, run `memory-hub init --team personal --path ./repo-mem`. Confirm the hub matches the architecture above.

### Phase 3 — Implement `status`, `add`, `promote`, `sync`
1. Build `lib/hub.ts` (resolve `.memory-hub`, walk up for hub root, validate JSON shape).
2. Build `lib/git.ts` (status, current SHA, ahead/behind, fetch, pull-ff-only, add, commit; reject on dirty tree where required).
3. Build `lib/wiki.ts` (frontmatter generation, `log.md` append, `index.md` regeneration from a directory walk).
4. Implement `status`, `add`, `promote`, `sync` using these libs.
5. Tests for each command against temp-dir fixtures with real git.
6. **Dogfood**: in a real consuming repo, `memory-hub link --hub ../repo-mem`, `memory-hub add concept memory-architecture`, `memory-hub promote memory/concepts/memory-architecture.md`, `memory-hub sync`. Confirm all six commands work end-to-end.

### Phase 4 — Validate cross-domain isolation
1. Run `memory-hub init --team payments` to create `~/development/team-payments-memory/`.
2. `memory-hub link --hub ../team-payments-memory` from a different consuming repo.
3. Add distinct facts to each hub (one to `repo-mem`, one to `team-payments-memory`).
4. From a Claude Code session in a `repo-mem`-linked repo, ask a question only answerable from the payments hub. Confirm the agent does NOT find it. Repeat the inverse.

### Phase 5 — Publish
1. Pick a package name (e.g., `@geekair/memory-hub-cli` or unscoped `memory-hub-cli`).
2. `npm publish --dry-run`, then publish.
3. Update READMEs to document `npx memory-hub <cmd>` invocation.

### Phase 6 — Deferred (future iterations)
- `search` and `index` commands (Memsearch integration).
- `doctor`, `unlink`, `pin`.
- Hub push policy / governance docs.
- CI for the CLI repo.

## Verification
- After Phase 1: `npx memory-hub --help` lists all six v1 commands as no-op stubs.
- After Phase 2: `memory-hub init --team personal --path ~/development/repo-mem` produces a hub with the full wiki scaffold and an initial commit. `memory-hub link --hub ~/development/repo-mem` in a sibling repo creates `.memory-hub`, `memory/`, and `CLAUDE.md`.
- After Phase 3: all six commands work against real git in temp-dir tests; manual dogfood walks the full add → promote → sync loop without manual editing.
- After Phase 4: cross-domain isolation test passes — payments-team facts never appear in `repo-mem`-linked sessions, and vice versa.
- After Phase 5: `npx memory-hub@latest init --team <foo>` works on a clean machine.

## Critical files this plan creates
- `~/development/memory-hub-cli/` — new repo (CLI tool source).
  - `package.json`, `tsconfig.json`, `biome.json`, `bin/memory-hub.js`
  - `src/cli.ts`, `src/commands/{init,link,status,add,promote,sync}.ts`
  - `src/lib/{hub,git,wiki,paths}.ts`
  - `src/templates/{AGENTS,CLAUDE,README,gitignore}.md.ts` + `src/templates/wiki/`
  - `tests/commands/*.test.ts`
- `~/development/repo-mem/` — produced by `memory-hub init --team personal` in Phase 2.
- `~/development/team-payments-memory/` — produced by `memory-hub init --team payments` in Phase 4.
- `<consuming-repo>/.memory-hub`, `<consuming-repo>/memory/`, `<consuming-repo>/CLAUDE.md` — produced by `memory-hub link` in consuming repos.

## What this plan does NOT do
- No federation across hubs — strong isolation is the goal.
- No automated promotion; `promote` is always an explicit command.
- No shared team-server backend; each hub is local-first.
- No `search` or `index` in v1 — Memsearch integration is deferred.
- No CI, hooks, or MCP integration in this phase.
- No GUI; CLI-only.

## Decisions locked in
- Architecture: per-team hub repos, JSON `.memory-hub` pointer, naming `team-<name>-memory` (with `repo-mem` as the "personal" exception).
- CLI runtime: **Node + TypeScript**, distributed via npm (`npx memory-hub`).
- CLI source lives in a **new standalone repo** `memory-hub-cli`, not inside any hub.
- v1 command set: **init, link, status, add, promote, sync**. `search` and `index` deferred to v2.
- Binary names: `memory-hub` (canonical) and `mh` (alias).
- CLI framework: `commander`. Tests: `vitest`. Lint/format: `biome`. Git access: shell out to `git`.
