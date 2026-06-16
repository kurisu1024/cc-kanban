---
description: Render the kanban board and regenerate board.md.
argument-hint: "[--epic PREFIX]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and read all item files.
- Build the board: one section per configured column, items grouped by epic when
  `settings.groupBoardByEpic`.
- If `$ARGUMENTS` contains `--epic PREFIX`, filter to that epic.
- Print the ASCII board (with the priority-glyph legend) **and** write the same
  content as markdown tables to `board.md`.
