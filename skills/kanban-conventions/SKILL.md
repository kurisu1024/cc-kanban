---
name: kanban-conventions
description: Core conventions for the kanban plugin — Obsidian Kanban board resolution, card grammar, id derivation, column reading, and safe edits. Every /kanban:* command applies these.
---

# Kanban Conventions (Obsidian Kanban-native)

Shared rules for all `/kanban:*` commands. The **Obsidian Kanban board `.md`
file is the sole source of truth** — there is no separate store, no per-item
files, no generated board. State lives in the board file.

## 1. Resolve the board
- `--board <name|path>`: use that board. A bare name resolves to
  `~/bigbrain/boards/<name>.md`.
- Else: if exactly one `*.md` board exists in `~/bigbrain/boards/`, use it.
- Else (none or many): list candidates and ask which, or offer `init`.
- A board is a `*.md` file whose YAML frontmatter contains `kanban-plugin: board`.

## 2. Board file anatomy
```
---
kanban-plugin: board
cc-kanban-prefix: PP        # id prefix for this board
---

## Backlog
- [ ] [PP-001] Story title #epic/slug #p/high #sz/M
  - [ ] a sub-issue

## Todo
## Doing
## Review
## Done

%% kanban:settings
```
{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}
```
%%
```
- **Columns** are the `##` headings, in file order. Read them from the file —
  never assume a fixed set beyond what `init` writes.
- The `%% kanban:settings %%` block is **always last**; preserve it verbatim on
  every write.

## 3. Card grammar
`^(\s*)- \[( |x|X)\] (?:\[([A-Z]{2,4}-\d{3,})\] )?(.*)$`
- **Story card:** no indent, has an `[ID]` token.
- **Sub-issue:** indented ≥ 2 spaces, no `[ID]`, belongs to the nearest story above.
- **Tags in text:** `#epic/<slug>` · `#p/high|med|low` · `#sz/S|M|L` · `#blocked` · `#paused`.
- **Done** cards live under `## Done` and are checked `- [x]`.

## 4. ID derivation (stateless)
- Prefix = frontmatter `cc-kanban-prefix`.
- New id = max existing `PREFIX-NNN` on the board + 1 (else 1), zero-padded to 3.
- Never reuse an id within one operation. (A deleted id may later recur — this is
  accepted; we keep no high-water mark.)

## 5. Editing rules
- Edits are **line-scoped and minimal**: insert/move/retag only the affected
  line(s). Never reflow or reformat the whole file.
- Preserve YAML frontmatter and the settings block on every write.
- `move <ID> <column>` relocates the card line (and its indented sub-issues)
  under the target `## Column`; set `- [x]` iff the target is `Done`, else `- [ ]`.

## 6. Read/render
- `board` parses the file and prints an ASCII summary grouped by column, with a
  legend: priority `◆ high · ▣ med · ○ low`, `⛔ blocked · ⏸ paused`. It writes
  nothing.

## 7. Safety
- `init` never clobbers an existing board.
- `import` is idempotent: skip a source that maps to an `[ID]` already present.
- Never delete a card unless explicitly asked.
- A missing/malformed board → offer `init` instead of erroring hard.
