---
description: Scaffold an Obsidian Kanban board file (boards/<name>.md).
---

Apply the `kanban-conventions` skill. Create an Obsidian Kanban board.

Args: `<name>` (board base name; default `board`), `--prefix XYZ` (id prefix;
default = uppercase of the first two alphanumerics of `<name>`).

Steps:
1. Resolve target path `~/bigbrain/boards/<name>.md`. If it already exists, STOP
   and report the path (never clobber).
2. Ensure `~/bigbrain/boards/` exists.
3. Write the board skeleton exactly:
   - frontmatter `kanban-plugin: board` + `cc-kanban-prefix: <PREFIX>`
   - the five headings `## Backlog`, `## Todo`, `## Doing`, `## Review`, `## Done`
   - the `%% kanban:settings %%` block with
     `{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}`
     as the **last** element.
4. Confirm: print the created path and tell the user to open it in Obsidian.
