---
description: Show one card with its sub-issues and tags.
---

Apply the `kanban-conventions` skill. Show a single card.

Args: `<ID>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Find `[<ID>]`. Print its column, full text, parsed tags
   (epic/priority/size/flags), and its sub-issues with checkbox state.
3. If not found, report and suggest `backlog`.
