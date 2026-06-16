---
description: Show a single item (epic, story, or issue) by id.
argument-hint: "<ID> [--all] [--global|--project]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and find the item file by id (search `epics/`, `stories/`,
  `issues/`). With `--all`, search **both** stores and show the matching item's
  scope badge (`🌐`/`📁`); if the same id exists in both, show both and label
  each. (`--global`/`--project` select a single store — skill §1.)
- Print a tidy summary of its frontmatter, then its body. For an epic, also list
  its child stories; for a story, list its child issues.
- If not found, suggest the closest matching ids.
