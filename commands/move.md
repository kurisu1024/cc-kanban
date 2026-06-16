---
description: Move an item between columns, or between stores with --to.
argument-hint: "<ID> <column>  |  <ID> --to global|project"
---

Apply the `kanban-conventions` skill.

**Column move** — `$ARGUMENTS` is `<ID> <column>`:

- Validate the id exists and the column is one of `config.json.columns`; if not,
  stop and show the valid values.
- Update the item's `status` and bump `updated` to today.
- Regenerate `board.md`.
- Confirm the move (`<old> → <new>` column).

**Cross-store move** — `$ARGUMENTS` is `<ID> --to global|project`:

- Follow skill §8 (Cross-store moves): locate the item across both stores,
  reallocate its id from the **target** store's counter, keep or detach the
  `epic`/`parent` link by whether the referent exists in the target, and — for an
  epic — offer to bring its child stories along.
- Write into the target store, remove from the source, bump `updated`, and
  regenerate **both** stores' `board.md`.
- Confirm the remap and any detached links
  (e.g. `📁 CTX-003 → 🌐 CTX-005 (epic link detached)`).
