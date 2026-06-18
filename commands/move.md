---
description: Move a card to another column (status change).
---

Apply the `kanban-conventions` skill. Relocate a card between columns.

Args: `<ID>` (required), `<column>` (required; must match an existing `##`
heading, case-insensitive), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Read the `##` headings; validate `<column>` matches one. If not, list valid
   columns and STOP.
3. Find the story card line `[<ID>]` and capture its indented sub-issues (the
   contiguous deeper-indented `- [ ]` lines beneath it).
4. Remove the card + its sub-issues from their current column.
5. Append them as the last card under the target `## <column>`.
6. Set the card's checkbox: `- [x]` iff target is `Done`, else `- [ ]`.
   (Leave sub-issue checkboxes unchanged.)
7. Preserve frontmatter and the settings block. Confirm: `<ID> → <column>`.
