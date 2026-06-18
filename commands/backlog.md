---
description: List/filter Backlog (and Todo) cards.
---

Apply the `kanban-conventions` skill. List near-term cards.

Args: `--epic <slug>`, `--priority high|med|low`, `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Collect cards under `## Backlog` and `## Todo`. Apply `--epic`/`--priority`
   filters by tag.
3. Print them grouped by column, sorted by priority (high→low), each as
   `[ID] text  #tags`.
