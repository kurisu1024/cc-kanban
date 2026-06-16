---
description: Move an item to another column.
argument-hint: "<ID> <column>"
---

Apply the `kanban-conventions` skill.

- Parse `$ARGUMENTS` as `<ID> <column>`.
- Validate the id exists and the column is one of `config.json.columns`; if not,
  stop and show the valid values.
- Update the item's `status` and bump `updated` to today.
- Regenerate `board.md`.
- Confirm the move (`<old> → <new>` column).
