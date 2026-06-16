---
description: Scaffold a kanban store (.claude/.kanban/) with config.json and an empty board.
argument-hint: "[--global]"
---

Apply the `kanban-conventions` skill.

Initialize a kanban store:

1. Resolve the target dir: `~/.claude/.kanban/` if `$ARGUMENTS` contains
   `--global`, else `<cwd>/.claude/.kanban/`.
2. If it already exists, report its location and item counts, then stop — do not
   clobber.
3. Otherwise create `epics/`, `stories/`, `issues/`, a `config.json` with the
   default columns / counters / settings, and an empty `board.md`.
4. Print the created location and the column list.
