---
description: Recommend what to work on next.
argument-hint: "[--epic PREFIX] [--global|--project]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and consider items in `backlog` / `todo` (optionally filtered
  by `--epic`).
- Rank by `priority` (high > medium > low), then smaller `size` first, then fewer
  unmet dependencies.
- Present the top 3–5 with a one-line rationale each, and recommend one to start.
- Offer to `/kanban:move` the chosen item to `doing`.
