---
description: Declare an epic tag for use on cards (epics are #epic/<slug> tags).
---

Apply the `kanban-conventions` skill. Epics are not files or columns — they are
`#epic/<slug>` tags applied to story cards.

Args: `<slug>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Normalize `<slug>` to kebab-case.
3. Report the tag to use — `#epic/<slug>` — and how to apply it: via
   `addstory --epic <slug>` or `groom`. Mention `backlog --epic <slug>` filters
   by it. (No file is created; the epic "exists" once a card carries the tag.)
