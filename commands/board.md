---
description: Render the kanban board and regenerate board.md.
argument-hint: "[--epic PREFIX] [--all] [--global|--project]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and read all item files. (`--global`/`--project` select which
  store; see skill §1.)
- Build the board: one section per configured column, items grouped by epic when
  `settings.groupBoardByEpic`.
- If `$ARGUMENTS` contains `--epic PREFIX`, filter to that epic.
- If `$ARGUMENTS` contains `--all`, render the **merged** view across both stores
  with scope badges (skill §6) and print only — do **not** write `board.md`.
- Otherwise print the ASCII board (with the priority-glyph legend) **and** write
  the same content as markdown tables to that store's `board.md`.
