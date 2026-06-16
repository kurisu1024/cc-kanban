---
description: Show a single item (epic, story, or issue) by id.
argument-hint: "<ID>"
---

Apply the `kanban-conventions` skill.

- Resolve the store and find the item file by id (search `epics/`, `stories/`,
  `issues/`).
- Print a tidy summary of its frontmatter, then its body. For an epic, also list
  its child stories; for a story, list its child issues.
- If not found, suggest the closest matching ids.
