---
description: Adopt existing markdown into board cards (idempotent).
---

Apply the `kanban-conventions` skill. Import markdown into a board.

Args: `<source>` (a file or directory of markdown), `--board <name|path>`
(create with `init` if absent), `--epic <slug>` (default epic for imported
cards), `--col <Column>` (default `Backlog`).

Steps:
1. Resolve or `init` the board.
2. Read `<source>`:
   - A directory of per-item files: each file → one story card (title from the
     first `#` heading or filename; map any `priority`/`size` frontmatter to
     `#p/`/`#sz/`).
   - A single structured page (e.g. a wiki status/open-questions page): map
     sections to columns where the section name implies status (e.g.
     "Recently done"→Done, "Next up"→Todo, "In progress"→Doing,
     unchecked bullets under "Open questions"→Backlog, checked `[x]`→Done).
3. **Idempotency:** before adding, derive a stable slug per source item; skip any
   item already represented on the board (match by title text). Never duplicate.
4. Assign ids via skill §4 as you add. Apply `--epic`/tags.
5. Insert cards under their mapped columns (or `--col`). Preserve the settings
   block.
6. Report: counts added per column, and anything skipped.
