---
description: List backlog items, optionally filtered.
argument-hint: "[--epic PREFIX] [--label X] [--priority high|medium|low]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and read items whose `status` is `backlog`.
- Apply any filters from `$ARGUMENTS` (`--epic`, `--label`, `--priority`).
- Print a compact list — `id · priority · size · title · epic` — sorted by
  priority (high → low), then size (S → L).
