---
description: Add a new story to the backlog.
argument-hint: "<description> [--epic PREFIX] [--priority high|medium|low] [--size S|M|L] [--global|--project]"
---

Apply the `kanban-conventions` skill. Ensure the store exists (init if needed).

Create a story from: $ARGUMENTS

- Parse the free-text description and optional flags (`--epic`, `--priority`,
  `--size`).
- If `--epic` is given, use that epic's prefix + counter for the id; otherwise
  use a default `STORY` prefix/counter (create it in `idCounters` if absent) and
  leave `epic` empty.
- Allocate the next id deterministically (bump the counter).
- Write `stories/<ID>-<kebab-title>.md` with `status` = `settings.defaultStatus`
  (backlog), today's `created`/`updated`, and `priority`/`size` from the flags or
  defaults (priority `medium`, size `M`). Seed a short body with a
  `## Acceptance Criteria` stub.
- Regenerate `board.md`.
- Print the new id, title, epic, and status.
