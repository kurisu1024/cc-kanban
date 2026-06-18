---
description: Print an ASCII summary of a board (read-only). --open opens it in Obsidian.
---

Apply the `kanban-conventions` skill. Render a board to the terminal.

Args: `--epic <slug>` (filter), `--open`, `--board <name|path>`.

Steps:
1. Resolve the board (skill §1). If `--open`, open it via `obsidian://open?path=`
   (URL-encode the vault-relative path) and stop.
2. Parse columns (`##` headings) and their cards. Optionally filter to
   `#epic/<slug>`.
3. Print one section per column; each card on a line as
   `<glyph> [ID] text` using priority glyphs `◆ high · ▣ med · ○ low`
   (and `⛔` if `#blocked`, `⏸` if `#paused`). Show sub-issue counts as `(n/m)`.
4. Print a legend. Write NOTHING to disk.
