---
description: Set priority/size/epic tags on cards collaboratively.
---

Apply the `kanban-conventions` skill. Groom cards by editing their tags.

Args: `--epic <slug>` (limit to one epic), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1). Collect story cards (optionally filtered to
   `#epic/<slug>`).
2. For each card lacking `#p/` or `#sz/`, propose values and ask the user
   (batch the questions). Respect existing tags unless told to change.
3. Apply confirmed tags by rewriting only that card's text (normalize order:
   `#epic/… #p/… #sz/…`). Do not touch other lines.
4. Summarize what changed.
