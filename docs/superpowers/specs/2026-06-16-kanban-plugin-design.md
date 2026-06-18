# Spec — `kanban` plugin for Claude Code

- **Date:** 2026-06-16
- **Status:** approved (design) → ready for implementation plan
- **Build location:** new git repo `~/workspaces/personal/cc-kanban/`, installed
  into Claude Code via local plugin path
- **Related:** the CC-experience backlog in `~/.claude/stories/` (12 stories the
  plugin will adopt as its first board)

## 1. Problem & Goal

We just hand-authored a 12-story backlog as loose markdown files. There is no
tooling to add, move, prioritize, or view that work — every operation is manual
file editing. **Goal:** a lightweight Claude Code plugin that manages an
agile/kanban backlog of epics, stories, and issues as markdown files, driven by
`/kanban:*` slash commands, with a generated board view. It must adopt the
existing stories with zero reformatting.

**Non-goals (YAGNI):** sprints/velocity, time tracking, multi-user assignment,
a web UI, external sync (Jira/GitHub Issues), or a long-running daemon.

## 2. Decisions (locked)

| Fork | Decision |
|------|----------|
| Location | **Both** — use a project's `.claude/.kanban/` when present, else global `~/.claude/.kanban/` |
| Storage | **Markdown file per item + generated `board.md` index** |
| Hierarchy | **Epics → Stories → Issues** (full, with parent links) |
| Columns | `backlog → todo → doing → review → done` (configurable) |
| Runtime | **None** — commands are prompt templates driving file ops via one shared skill |
| Build location | `~/workspaces/personal/cc-kanban/` (git repo, local-path install) |

## 3. Plugin Layout

```
cc-kanban/
├── .claude-plugin/plugin.json        # name: kanban, version, description
├── commands/                         # one .md per slash command (prompt templates)
│   ├── init.md      addepic.md   addstory.md   addissue.md
│   ├── board.md     backlog.md   move.md       story.md
│   └── groom.md     pick.md      import.md
├── skills/
│   └── kanban-conventions/SKILL.md   # shared core: dir resolution, schema, ids, board format
└── README.md
```

The **`kanban-conventions` skill is the single source of truth**. Every command
file is thin and references it, so all commands resolve the store, generate ids,
and render the board identically (DRY). Logic is model-driven file manipulation —
no compiled code, no dependencies.

## 4. Data Store

Resolved location: **project `<repo>/.claude/.kanban/` if it exists, else global
`~/.claude/.kanban/`.** (`/kanban:init` creates whichever the user targets.)

```
.claude/.kanban/
├── config.json        # { columns[], idCounters{}, settings }
├── board.md           # GENERATED kanban view (the index) — never hand-edited
├── epics/   EPIC-001-context-memory-iq.md
├── stories/ CTX-001-warm-start.md
└── issues/  ISS-001-warm-start-git-summarizer.md
```

### Item schema (frontmatter)

A superset of the existing 12 stories' frontmatter, so import is loss-free:

```yaml
---
id: CTX-001            # stories: epic-prefixed · epics: EPIC-NNN · issues: ISS-NNN
type: story            # epic | story | issue
epic: CTX              # story → parent epic prefix; (epics omit)
parent: CTX-001        # issue → parent story id; (stories/epics omit)
status: backlog        # backlog | todo | doing | review | done
priority: high         # high | medium | low
size: S                # S | M | L
labels: [harness]
created: 2026-06-16
updated: 2026-06-16
---
# CTX-001 · Warm Start …
body (problem / solution / acceptance criteria / …)
```

### config.json

```json
{
  "columns": ["backlog", "todo", "doing", "review", "done"],
  "idCounters": { "EPIC": 1, "CTX": 4, "TUI": 4, "DAT": 4, "ISS": 0 },
  "settings": { "defaultStatus": "backlog", "groupBoardByEpic": true }
}
```

## 5. Commands

All namespaced `/kanban:…`.

| Command | Behavior |
|---------|----------|
| `/kanban:init [--global]` | scaffold the store + `config.json`. Auto-runs on first add if missing. `--global` forces `~/.claude/.kanban/` even inside a repo. |
| `/kanban:addepic <name> [--prefix XYZ]` | create epic file; assign `EPIC-NNN` and a story prefix (derived from name or `--prefix`). |
| `/kanban:addstory <desc> [--epic CTX] [--priority --size]` | create story in `backlog`; auto-id from the epic's prefix; bump counter. |
| `/kanban:addissue <desc> --story CTX-001` | create child issue (`ISS-NNN`), `parent` set. |
| `/kanban:board [--epic CTX]` | regenerate `board.md` and print the ASCII board. |
| `/kanban:backlog [--epic --label --priority]` | list backlog items, filtered. |
| `/kanban:move <id> <column>` | change `status`, touch `updated`, regen board. Validates column. |
| `/kanban:story <id>` | print one card (frontmatter + body). |
| `/kanban:groom` | interactive pass: walk backlog, set priority/size/epic/links collaboratively. |
| `/kanban:pick` | recommend next work from `status` + `priority` × `size` × dependencies. The "what should we do together" command. |
| `/kanban:import <dir>` | adopt existing markdown items (e.g. `~/.claude/stories/`); add missing `status`, infer `epic` from id prefix; **idempotent** (skip existing ids). |

## 6. Board Rendering (`/kanban:board`)

```
KANBAN · ~/.claude/.kanban (global) · 12 items
─────────────────────────────────────────────────────────────
BACKLOG (10)        TODO (1)     DOING (1)    REVIEW (0)  DONE (0)
CTX-002 /recall ◆   CTX-001 ▣    DAT-002 ⟳
DAT-001 emitter ◆
TUI-001 /usage  ○
…
─────────────────────────────────────────────────────────────
epics: CTX(4) · TUI(4) · DAT(4)     ◆ high  ▣ med  ○ low
```

`board.md` persists the same view as markdown (a table per column) so it renders
in any markdown viewer and diffs cleanly in git.

## 7. Error Handling & Edge Cases

- **No store yet** → auto-run `init` before the first write.
- **Unknown column** in `move` → reject with the valid column list.
- **Unknown / ambiguous id** → reject; suggest closest matches.
- **Id collisions** → counters in `config.json` are the single allocator; never
  reuse.
- **`import` re-run** → idempotent; ids already present are skipped, not
  duplicated.
- **Project vs global ambiguity** → resolution is deterministic (project store
  wins when `<repo>/.claude/.kanban/` exists); `--global` overrides.
- **`board.md` is generated** → commands always regenerate it; it is never the
  source of truth (the item files are).

## 8. First Win (acceptance smoke test)

```
/kanban:import ~/.claude/stories
/kanban:board
```
→ the 12 existing stories appear on the board, grouped into CTX / TUI / DAT epics,
all in `backlog`, with nothing reformatted by hand.

## 9. Acceptance Criteria

- [ ] `/kanban:init` scaffolds `.claude/.kanban/` (project or `--global`) with
      `config.json` and an empty `board.md`.
- [ ] `/kanban:addstory "x" --epic CTX` creates a `backlog` story with a unique
      epic-prefixed id and bumps the counter.
- [ ] `/kanban:addepic` and `/kanban:addissue` create correctly-linked items.
- [ ] `/kanban:move CTX-001 doing` updates status + `updated` and regenerates the
      board.
- [ ] `/kanban:board` prints the ASCII board and writes `board.md` matching the
      item files.
- [ ] `/kanban:backlog` filters by epic/label/priority.
- [ ] `/kanban:pick` recommends next work using priority/size/deps/status.
- [ ] `/kanban:import ~/.claude/stories` adopts the 12 stories idempotently with
      no manual reformatting.
- [ ] Store resolution is deterministic (project-first, global fallback,
      `--global` override).
- [ ] All commands share the `kanban-conventions` skill (no duplicated logic).

## 10. Hardening (post-MVP, not in scope)

- Back id-allocation + `board.md` regeneration with a small node script for
  determinism (commands shell out to it instead of doing arithmetic in-prompt).
- `/kanban:export` to feed claude-stats (ties to the DAT-001 run-event work).
- Dependency links (`dependsOn`) + `/kanban:pick` honoring them as a real DAG.

## 11. Open Questions (carry into planning)

- Should `/kanban:pick` and `/kanban:groom` be model-judgement only, or also
  surface a deterministic ranking the model then explains?
- Do we want a bare-command alias set (`/addstory`) in addition to the namespaced
  `/kanban:addstory`, or keep everything namespaced to avoid collisions?
