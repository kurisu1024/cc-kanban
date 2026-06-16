---
description: Scaffold a kanban store (.claude/.kanban/) with config.json and an empty board.
argument-hint: "[--global]"
---

Apply the `kanban-conventions` skill.

Initialize a kanban store:

1. Resolve the target dir and scope: `~/.claude/.kanban/` (scope `global`) if
   `$ARGUMENTS` contains `--global`, else `<cwd>/.claude/.kanban/` (scope
   `project`).
2. If it already exists, report its location, scope, and item counts, then stop —
   do not clobber.
3. Otherwise create `epics/`, `stories/`, `issues/`, a `config.json` with the
   resolved `scope` plus the default columns / counters / settings, and an empty
   `board.md`.
4. Print the created location, scope, and the column list.
