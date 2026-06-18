---
description: Add a sub-issue (nested checkbox) under a story card.
---

Apply the `kanban-conventions` skill. Add a sub-issue to a story.

Args: `<desc>` (required), `--story <ID>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Find the story card line whose id token is `[<ID>]`. If not found, STOP and
   report.
3. Insert `  - [ ] <desc>` (two-space indent) immediately after the story card's
   existing sub-issues (or right after the card line if it has none).
4. Confirm: print the parent id and the sub-issue text.
