---
description: Adopt existing markdown items into the kanban store (idempotent).
argument-hint: "<source-dir>"
---

Apply the `kanban-conventions` skill. Ensure the store exists (init if needed).

Import from the directory in `$ARGUMENTS` (e.g. `~/.claude/stories`):

- For each markdown file with frontmatter `id` / `type`, copy it into the subdir
  matching its `type` (`epics` / `stories` / `issues`, defaulting to `story`).
- Add `status: backlog` if missing. Infer `epic` from the id prefix (e.g.
  `CTX-001` → prefix `CTX`) and register that prefix in `config.json.idCounters`
  at the highest-seen number.
- **Skip any id that already exists** in the store (idempotent — never
  duplicate).
- If prefixes imply epics that have no epic file, offer to create stub epic files
  (ask first).
- Regenerate `board.md` and report what was imported vs skipped.
