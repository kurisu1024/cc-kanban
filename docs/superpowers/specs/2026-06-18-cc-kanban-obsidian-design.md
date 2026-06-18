# cc-kanban v2 — Obsidian Kanban-native design

**Date:** 2026-06-18
**Status:** Approved (design)
**Author:** Chris Clark (with Claude)

## Problem

cc-kanban v1 stores a backlog as **many small markdown files** (`epics/ stories/
issues/`) inside a `.claude/.kanban/` store and renders a **read-only generated
`board.md`**. The user has installed the [Obsidian Kanban plugin]
(mgmeyers/obsidian-kanban), whose model is the inverse: **one markdown file per
board *is* the source of truth** — columns are `##` headings, cards are `- [ ]`
list items, and the plugin reads/writes that single file in place.

The two models are fundamentally at odds. Maintaining both means a fragile
two-way sync. The goal is to make cc-kanban **strictly Obsidian Kanban-native**:
the board file is the only state, and cc-kanban becomes the agent-side interface
that creates, reads, and maintains Obsidian-format boards.

## Decisions (locked during brainstorming)

1. **Data model:** the Obsidian Kanban board `.md` file is the *sole* source of
   truth. No per-item file store, no `config.json`, no generated `board.md`, no
   sync layer.
2. **Hierarchy:** one board; columns = status lanes; cards = stories. Epics are
   `#epic/<slug>` tags, not files. Issues are nested sub-checkboxes under a story.
3. **Location:** boards live at `~/bigbrain/boards/<name>.md` (inside the Obsidian
   vault so the plugin renders them; versioned with the wiki).
4. **First board:** a fresh `boards/poolproof.md` seeded from the bigbrain
   poolproof wiki. The legacy global store and `~/.claude/stories/` are deleted.
5. **Command namespace:** keep `/kanban:*` (no churn); rewrite behavior. Version
   bumps to **2.0.0** (breaking).

## Board file format

Boards are authored to the Obsidian Kanban serialization. A board has YAML
frontmatter with `kanban-plugin: board`, one `##` heading per column, `- [ ]`
cards under each, and a trailing `%% kanban:settings %%` block. Example
`boards/poolproof.md`:

````markdown
---
kanban-plugin: board
---

## Backlog

- [ ] [PP-004] Resume-reconcile: recompute route when a hold is lifted #epic/route-reconcile #p/med #sz/M
- [ ] [PP-005] Confirm reconcile covers already-generated future-week routes #epic/route-reconcile #p/med #sz/S

## Todo

- [ ] [PP-003] Decide on resume-reconcile approach + hook point #epic/route-reconcile #p/high #sz/S

## Doing

## Review

## Done

- [x] [PP-001] Suspend → route-map refresh client fix #epic/suspend-flow #p/high #sz/S
- [x] [PP-002] Day-of-hold server reconcile (7af90770) #epic/suspend-flow #p/high #sz/M

%% kanban:settings
```
{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}
```
%%
````

### Card conventions (all inline-native, so Obsidian renders them as real tags)

| Element | Convention | Notes |
|---|---|---|
| **ID** | leading `[PP-NNN]` token | Per-board prefix derived from board name; counter = scan board, `max+1`. No external state. |
| **Epic** | `#epic/<slug>` | Replaces v1 epic files. Nested tag so Obsidian groups/filters by it. |
| **Priority** | `#p/high` · `#p/med` · `#p/low` | |
| **Size** | `#sz/S` · `#sz/M` · `#sz/L` | |
| **Blocked/paused** | `#blocked` or `#paused` | Stays in its column; tag signals state. |
| **Issue** | nested `- [ ]` under a story card | Sub-checkboxes; Obsidian keeps them in the card body. |
| **Done** | card under `## Done`, checked `- [x]` | `move <ID> done` toggles the checkbox. |

**ID prefix:** derived from the board name (`poolproof` → `PP`). Stored nowhere;
re-derived by scanning. Default prefix = first letters / a 2–3 char uppercase
slug. `init` may accept `--prefix`.

**Why no created/updated per card:** board = truth and Obsidian does not persist
per-card timestamps in this format. We rely on git history for the file. Optional
due dates use Obsidian's native `@{YYYY-MM-DD}` only when a card needs one (YAGNI).

### Columns

Keep v1's flow as Obsidian headings: **Backlog · Todo · Doing · Review · Done**.
Column set is read from the board's own `##` headings (not a config file). `init`
writes these five; users may add/rename headings in Obsidian and cc-kanban honors
whatever headings the file currently has.

## Command surface (`/kanban:*`, rewritten for board-as-truth)

All commands resolve a **board file** instead of a store. Resolution order:
explicit `--board <name|path>` → single board in `~/bigbrain/boards/` → if
multiple, prompt. (cwd-based project resolution from v1 is dropped; boards are
vault-global.)

| Command | v2 behavior |
|---|---|
| `init [name] [--prefix XYZ]` | Create `boards/<name>.md` with frontmatter, 5 column headings, settings block. **Idempotent** — never clobber an existing board. |
| `addstory <desc> [--epic s] [--priority p] [--size s] [--col Backlog]` | Append `- [ ] [ID] desc #epic/.. #p/.. #sz/..` under the target column (default Backlog). ID auto-derived. |
| `addissue <desc> --story <ID>` | Insert a nested `- [ ]` under the matching story card. |
| `addepic <slug>` | Lightweight — epics are tags. Records the epic slug (e.g. in a board legend comment) and is available to `--epic`. No file created. |
| `move <ID> <column>` | Find the card line by `[ID]`; relocate it under the target `## Column`; toggle `[x]` when the column is Done (and uncheck when leaving Done). |
| `board [--open]` | Parse the board file → print an ASCII summary grouped by column (priority glyphs + legend). `--open` opens the file in Obsidian (`obsidian://open`). No longer writes anything. |
| `backlog [--epic s] [--priority p]` | List/filter Backlog (and Todo) cards by tag. |
| `story <ID>` | Show one card: title, tags, sub-issues, and any body lines. |
| `groom [--epic s]` | Interactively set `#p/` `#sz/` `#epic/` tags on cards (rewrites tags on card lines). |
| `pick [--epic s]` | Recommend the next card: highest `#p/` in Backlog/Todo, ties broken by smaller `#sz/`. |
| `import <dir>` | Adopt existing per-item markdown into cards (also the migration engine — see below). Idempotent: skip cards whose source maps to an `[ID]` already on the board. |

The `kanban-conventions` SKILL.md is rewritten to describe **this** format
(board resolution, card grammar, ID derivation, column reading, settings block,
safety). Command files stay thin and defer to it.

## Parsing & writing rules (the editing contract)

cc-kanban edits the board as a line-oriented markdown document:

- **Card line grammar:** `^(\s*)- \[( |x)\] \[(?<id>[A-Z]+-\d+)\] (?<text>.*)$`
  where trailing `#tag` tokens are metadata. Sub-issues are deeper-indented
  `- [ ]` lines until the next same-or-shallower card.
- **Column = region** between one `## Heading` and the next `##` (or the
  `%% kanban:settings` block).
- **Writes are minimal and scoped:** insert/move/edit only the affected line(s);
  never reformat the whole file; preserve the settings block verbatim.
- **ID derivation:** scan all `[PREFIX-\d+]` tokens for the board's prefix; next
  id = `max+1`, zero-padded to 3. Never reuse.
- **Safety:** never delete a card unless explicitly asked; keep frontmatter and
  the settings block intact; if the board file is missing, offer `init`.

## Migration + cleanup

### Seed `boards/poolproof.md` from the wiki (one-time, via `import` logic)

Source: `~/bigbrain/wiki/projects/poolproof/{status.md, open-questions.md}`.
Mapping:

| Wiki source | → Column | Example cards |
|---|---|---|
| `status.md` → *Recently done* | **Done** (`- [x]`) | PP-001 suspend→map-refresh client fix; PP-002 day-of-hold server reconcile |
| `status.md` → *Next up* | **Todo** | PP-003 decide on resume-reconcile approach |
| `status.md` → *In progress* | **Doing** | (clean tree at design time → none active) |
| `open-questions.md` (unchecked) | **Backlog** | PP-004 resume-reconcile; PP-005 future-week coverage; end-bounded suspensions |
| `open-questions.md` (checked `[x]`) | **Done** | resolved end-to-end reconcile verification |
| paused billing (`git stash`) | **Backlog** + `#paused` | a single "day-of-hold billing (paused)" card, left untouched |

Epics: `#epic/suspend-flow` (shipped work) and `#epic/route-reconcile` (open
follow-ups). Exact card text is drafted at implementation time from the wiki
prose; ids assigned sequentially `PP-001…`.

### Delete the legacy data

- **Delete** `~/.claude/.kanban/` (global store: `epics/ stories/ issues/`,
  `config.json`, `board.md` — the 12 CTX/TUI/DAT items).
- **Delete** `~/.claude/stories/` (the original source markdown for those items),
  **except**: **archive** `~/.claude/stories/**/2026-06-16-kanban-plugin-design.md`
  (the original plugin design doc) into
  `~/workspaces/personal/cc-kanban/docs/superpowers/specs/` before removing the
  directory, so the project's own history is preserved.
- **Do NOT touch** the project store at
  `~/workspaces/personal/claude-stats/.claude/.kanban/` — out of scope.

These deletions are authorized by the user ("delete the others"). They are
outside the bigbrain repo; perform them only after the poolproof board is seeded
and confirmed.

### Implementation target & sync

- Implement in the **dev repo** `~/workspaces/personal/cc-kanban` (git remote
  `kurisu1024/cc-kanban`). Commit there.
- The installed copy at `~/.claude/plugins/marketplaces/cc-kanban/` is a separate
  clone; after committing, **sync/reinstall** (e.g. `/plugin marketplace update`
  or reinstall `kanban@cc-kanban`) so `/kanban:*` reflects v2.
- The seeded `boards/poolproof.md` is committed in the **bigbrain** repo.

## Out of scope (YAGNI)

- Two-way sync / per-item file store (explicitly removed).
- Per-card created/updated timestamps.
- cwd/project-walk board resolution (boards are vault-global now).
- Cross-board "move between stores" (v1 promote/demote) — single vault, drop it.
- Migrating the CTX/TUI/DAT backlog (user chose a fresh poolproof board).
- Touching the claude-stats project store.

## Testing / verification

- **Format conformance:** open `boards/poolproof.md` in Obsidian → it renders as a
  Kanban board with the five lanes and seeded cards (manual check).
- **Round-trip:** `addstory` → card appears under Backlog in Obsidian; drag a card
  in Obsidian → `board`/`story` reflect the new column; `move <ID> done` → card
  shows `- [x]` under Done in Obsidian.
- **ID derivation:** adding cards yields strictly increasing, non-reused ids;
  deleting a card then adding does not reuse the removed id (max+1 over remaining
  — documented behavior).
- **Idempotent init/import:** re-running never clobbers or duplicates.
- **Safety:** a malformed/missing board offers `init` rather than erroring hard;
  the settings block survives every write.
```
