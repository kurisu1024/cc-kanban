---
description: Add a new epic (a group of stories).
argument-hint: "<name> [--prefix XYZ] [--global|--project]"
---

Apply the `kanban-conventions` skill. Ensure the store exists (init if needed).

Create an epic from: $ARGUMENTS

- Derive a story `prefix` from `--prefix`, or from the name (uppercase
  initials / leading letters). Ensure it is unique in `config.json.idCounters`.
- Allocate `EPIC-NNN` (bump the `EPIC` counter). Initialize the new prefix
  counter at 0.
- Write `epics/EPIC-NNN-<kebab-name>.md` with `type: epic`, the chosen `prefix`
  in frontmatter, `status: backlog`, and today's `created`/`updated`.
- Regenerate `board.md`.
- Print the epic id, name, and story prefix.
