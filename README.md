# cc-kanban

> Obsidian Kanban-native backlog for Claude Code — `/kanban:*` commands
> maintain board files as the single source of truth.

**Version 2.0.0**

## Overview

cc-kanban is a Claude Code plugin that manages a project backlog through
[Obsidian Kanban](https://github.com/mgmeyers/obsidian-kanban)-format board
files. The board file **is** the data — no separate store, no generated
`board.md`, no sync layer.

Boards live at `~/bigbrain/boards/<name>.md` inside your Obsidian vault.
Obsidian Kanban renders them visually; cc-kanban lets you manage them from
the Claude Code agent.

## Board format

```yaml
---
kanban-plugin: board
cc-kanban-prefix: PP
---
```

Columns are `##` headings. Cards are checklist items under those headings.

## Card grammar

```
- [ ] [PP-001] Story title #epic/slug #p/high #sz/M
  - [ ] a sub-issue
```

| Token | Meaning |
|---|---|
| `[PP-001]` | Story id (prefix from frontmatter, auto-incremented) |
| `#epic/<slug>` | Epic grouping tag |
| `#p/high\|med\|low` | Priority |
| `#sz/S\|M\|L` | Size estimate |
| `#blocked` / `#paused` | Status flags |
| nested `  - [ ]` | Sub-issue (2-space indent, no id) |
| `- [x]` under `## Done` | Completed card |

## Commands

| Command | Description |
|---|---|
| `/kanban:init [name] [--prefix XYZ]` | Create `boards/<name>.md` with 5 columns and settings block |
| `/kanban:addstory <desc> [--epic s] [--priority p] [--size s] [--col C]` | Append a story card |
| `/kanban:addissue <desc> --story <ID>` | Add a sub-issue under a story |
| `/kanban:addepic <slug>` | Declare an epic tag (no file created) |
| `/kanban:move <ID> <column>` | Relocate a card; toggles `[x]` when target is Done |
| `/kanban:board [--epic s] [--open]` | Print ASCII summary; `--open` opens in Obsidian |
| `/kanban:backlog [--epic s] [--priority p]` | List Backlog + Todo cards |
| `/kanban:story <ID>` | Show one card with sub-issues and tags |
| `/kanban:groom [--epic s]` | Set priority/size/epic tags collaboratively |
| `/kanban:pick [--epic s]` | Recommend the next card to work on |
| `/kanban:import <source> [--board b] [--epic s] [--col C]` | Adopt markdown into board cards (idempotent) |

## Board resolution

1. `--board <name|path>` — bare name → `~/bigbrain/boards/<name>.md`
2. Single `*.md` board in `~/bigbrain/boards/` → use it automatically
3. Multiple boards → list and prompt
4. No board → offer `init`

## Settings block

Every board ends with this block (Obsidian Kanban requires it):

```
%% kanban:settings
```json
{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}
```
%%
```

cc-kanban preserves this block verbatim on every write.

## Getting started

```bash
# Create a board
/kanban:init myproject --prefix MP

# Add some work
/kanban:addstory "Set up CI" --priority high --size S
/kanban:addstory "Write docs" --epic docs --priority med --size M
/kanban:addissue "Add GitHub Actions workflow" --story MP-001

# View the board
/kanban:board

# Move work through columns
/kanban:move MP-001 Doing
/kanban:move MP-001 Done
```

## Installation

Install via the Claude Code plugin marketplace:

```bash
/plugin install kanban@cc-kanban
```

Or clone the repo and symlink into your plugins directory.

## License

MIT
