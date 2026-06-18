---
description: Recommend the next card to work on.
---

Apply the `kanban-conventions` skill. Recommend the next card.

Args: `--epic <slug>`, `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Candidates = cards in `## Todo` then `## Backlog` (Todo first). Filter by
   `--epic` if given.
3. Rank by priority (high→low); break ties by smaller size (S<M<L); then file
   order. Recommend the top card and briefly say why.
4. Offer to `move <ID> Doing` for the chosen card (don't move without confirmation).
