---
description: List backlog items, optionally filtered.
argument-hint: "[--epic PREFIX] [--label X] [--priority high|medium|low] [--all] [--global|--project]"
---

Apply the `kanban-conventions` skill.

- Resolve the store and read items whose `status` is `backlog`.
  (`--global`/`--project` select which store; `--all` unions both — skill §1/§6.)
- Apply any filters from `$ARGUMENTS` (`--epic`, `--label`, `--priority`).
- Print a compact list — `id · priority · size · title · epic` — sorted by
  priority (high → low), then size (S → L). Under `--all`, prefix each row with
  its scope badge (`🌐`/`📁`).
