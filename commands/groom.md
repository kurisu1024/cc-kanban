---
description: Collaboratively groom the backlog — set priority, size, epic, and links.
argument-hint: "[--epic PREFIX]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and gather `backlog` items (optionally filtered by
  `--epic`).
- Walk them with the user a few at a time: propose `priority` / `size` / `epic` /
  `labels` and any obvious dependency links, and ask for confirmation or edits.
- Apply confirmed changes to the item files (bump `updated`), then regenerate
  `board.md`.
- Keep it interactive and incremental — never bulk-rewrite without confirmation.
