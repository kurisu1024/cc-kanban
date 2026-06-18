---
description: Add a story card to a board column (default Backlog).
---

Apply the `kanban-conventions` skill. Append a story card to a board.

Args: `<desc>` (required), `--epic <slug>`, `--priority high|med|low`,
`--size S|M|L`, `--col <Column>` (default `Backlog`), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1). If none, offer `init` and STOP.
2. Derive the new id (skill §4): prefix from frontmatter, max+1.
3. Build the card text:
   `- [ ] [<ID>] <desc>` then append, when provided:
   ` #epic/<slug>` ` #p/<priority>` ` #sz/<size>`.
4. Insert the line as the LAST card under the `## <Column>` heading (just before
   the next `##` or the settings block). Preserve everything else.
5. Confirm: print the new id, column, and tags.
