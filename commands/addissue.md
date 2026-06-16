---
description: Add a child issue under a story.
argument-hint: "<description> --story <STORY-ID>"
---

Apply the `kanban-conventions` skill. Ensure the store exists.

Create an issue from: $ARGUMENTS

- Require `--story <id>`; if it is missing or unknown, stop and report.
- Allocate `ISS-NNN` (bump the `ISS` counter).
- Write `issues/ISS-NNN-<kebab-title>.md` with `type: issue`,
  `parent: <STORY-ID>`, `status: backlog`, and today's `created`/`updated`.
- Regenerate `board.md`.
- Print the issue id and its parent story.
