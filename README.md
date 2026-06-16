# kanban — agile backlog for Claude Code

A lightweight Claude Code plugin that manages an agile/kanban backlog of
**epics → stories → issues** as markdown files, driven by `/kanban:*` slash
commands, with a generated board view. No runtime, no dependencies — commands
are prompt templates that drive file operations via one shared skill
(`kanban-conventions`).

## Store

Resolved **project-first, global fallback**:

- `<repo>/.claude/.kanban/` when it exists, else `~/.claude/.kanban/`
- `--global` forces the global store

```
.claude/.kanban/
  config.json        # columns, id counters, settings
  board.md           # generated kanban view
  epics/  stories/  issues/
```

Each item is one markdown file with frontmatter (`id`, `type`, `status`,
`priority`, `size`, `epic`/`parent`, …) plus a free body.

## Commands

| Command | Does |
|---------|------|
| `/kanban:init [--global]` | scaffold the store |
| `/kanban:addepic <name> [--prefix XYZ]` | new epic + story prefix |
| `/kanban:addstory <desc> [--epic P] [--priority] [--size]` | new story → backlog |
| `/kanban:addissue <desc> --story <ID>` | child issue |
| `/kanban:board [--epic P]` | render board + regenerate `board.md` |
| `/kanban:backlog [--epic --label --priority]` | filtered backlog list |
| `/kanban:move <ID> <column>` | change status |
| `/kanban:story <ID>` | show one item |
| `/kanban:groom [--epic P]` | collaboratively set priority/size/links |
| `/kanban:pick [--epic P]` | recommend next work |
| `/kanban:import <dir>` | adopt existing markdown (idempotent) |

Columns: `backlog → todo → doing → review → done` (configurable in
`config.json`).

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
