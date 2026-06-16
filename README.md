# kanban — agile backlog for Claude Code

A lightweight Claude Code plugin that manages an agile/kanban backlog of
**epics → stories → issues** as markdown files, driven by `/kanban:*` slash
commands, with a generated board view. No runtime, no dependencies — commands
are prompt templates that drive file operations via one shared skill
(`kanban-conventions`).

## Store

There are two stores — a **project** store per repo and one **global** store —
and commands act on whichever they resolve.

```
.claude/.kanban/
  config.json        # scope, columns, id counters, settings
  board.md           # generated kanban view
  epics/  stories/  issues/
```

Each item is one markdown file with frontmatter (`id`, `type`, `status`,
`priority`, `size`, `epic`/`parent`, …) plus a free body.

### Global vs project stores

Resolution picks one **active store** (the read/write target):

1. `--global` → the global store at `~/.claude/.kanban/`.
2. `--project` → the nearest project store (errors and offers `init` if none).
3. **Default:** walk **up** from the current directory for a `.claude/.kanban/`
   (git-style — so it resolves from any subfolder of the repo, not just the
   root). If none is found, fall back to the global store.

The walk-up stops at the filesystem root and never treats `~/.claude/.kanban/`
itself as a project store, so the global store is only ever hit via `--global`
or as the fallback. Each store keeps its own id counters, so the same id (e.g.
`CTX-001`) can exist in both independently.

**See everything at once.** Reads (`board`, `backlog`, `story`) accept `--all`
to show a **merged** view across both stores, badged `🌐` global / `📁` project.
`--all` is print-only and never rewrites either `board.md`.

**Move between stores.** `/kanban:move <ID> --to global|project` promotes or
demotes an item: it reallocates the id from the target store's counter, keeps the
`epic`/`parent` link if the referent exists there (otherwise detaches it with a
warning), and offers to bring an epic's child stories along.

## Commands

| Command | Does |
|---------|------|
| `/kanban:init [--global]` | scaffold the store |
| `/kanban:addepic <name> [--prefix XYZ]` | new epic + story prefix |
| `/kanban:addstory <desc> [--epic P] [--priority] [--size]` | new story → backlog |
| `/kanban:addissue <desc> --story <ID>` | child issue |
| `/kanban:board [--epic P] [--all]` | render board + regenerate `board.md` |
| `/kanban:backlog [--epic --label --priority] [--all]` | filtered backlog list |
| `/kanban:move <ID> <column>` | change status |
| `/kanban:move <ID> --to global\|project` | promote/demote across stores |
| `/kanban:story <ID> [--all]` | show one item |
| `/kanban:groom [--epic P]` | collaboratively set priority/size/links |
| `/kanban:pick [--epic P]` | recommend next work |
| `/kanban:import <dir>` | adopt existing markdown (idempotent) |

Most commands also accept `--global` / `--project` to pick the store explicitly
(default is project-first with global fallback). Columns:
`backlog → todo → doing → review → done` (configurable in `config.json`).

## Install (local)

This repo is also a single-plugin marketplace. From Claude Code:

```
/plugin marketplace add ~/workspaces/personal/cc-kanban
/plugin install kanban@cc-kanban
```

(Or via CLI: `claude plugin marketplace add ~/workspaces/personal/cc-kanban`
then `claude plugin install kanban@cc-kanban`.) Restart the session so the
`/kanban:*` commands load.

## First run

```
/kanban:import ~/.claude/stories   # adopt the existing 12-story backlog
/kanban:board                      # see them grouped into CTX / TUI / DAT epics
```

## Roadmap (post-MVP)

- Back id-allocation + board regeneration with a small node script for
  determinism.
- `dependsOn` links + a real DAG-aware `/kanban:pick`.
- `/kanban:export` to feed claude-stats analytics.
