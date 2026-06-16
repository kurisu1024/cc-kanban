# Dual-scope stores (global + project) — design

**Status:** approved 2026-06-16 · **Target:** kanban plugin v1.1

## Problem

The kanban plugin resolves exactly one store: the project store at
`<cwd>/.claude/.kanban/` if it exists, else the global store at
`~/.claude/.kanban/`. Two gaps:

1. The project store only resolves when cwd is *exactly* its parent dir — `cd`
   into any subfolder and the project store disappears, silently falling back to
   global.
2. There is no way to see project and global work together, and no way to move
   an item between stores.

## Architecture principle (unchanged)

All behavior lives in `skills/kanban-conventions/SKILL.md`. The 11 command files
stay thin (`Apply the kanban-conventions skill` + their specific job) and inherit
resolution for free. ~80% of this change is a single skill edit.

## A. Resolution & precedence (SKILL.md §1 rewrite)

Resolution yields an **active store** — the write target. Order:

1. `--global` flag → `~/.claude/.kanban`
2. `--project` flag → nearest project store via walk-up; if none, error and offer
   `init`
3. **default:** walk up from cwd for `.claude/.kanban`; first hit → that project
   store; else → global

**Walk-up rule:** ascend from cwd checking `<dir>/.claude/.kanban`, stopping at
`$HOME` and the filesystem root. `$HOME` itself is **excluded** from the walk so
that `~/.claude/.kanban` (the global store) can never be matched as a project
store. Global is only ever reached as the explicit `--global` target or the
final fallback.

## B. config.json scope self-id (SKILL.md §2)

```json
{ "scope": "project", "columns": [...], "idCounters": {...}, "settings": {...} }
```

`scope` is written at `init` (`"global"` under `--global`, else `"project"`).
`idCounters` stay **per-store and independent** — global `CTX-001` and project
`CTX-001` may coexist. They only ever appear together in the badged merged view,
where the scope badge disambiguates.

## C. Merged view (SKILL.md §6; board / backlog / story)

- `board`, `backlog`, and `story` accept `--all`: read **both** stores (project +
  global) and union the items.
- Each item is badged: 🌐 global · 📁 project. The legend gains these glyphs.
- Layout: column → scope subsection → epic grouping.
- **`--all` is print-only.** It never overwrites either store's `board.md`. Each
  store's `board.md` stays canonical and single-scope, so one store's board never
  absorbs the other's items.

## D. Cross-store move (move.md; SKILL.md §7)

New mode on the existing `move` command; the status-change behavior
(`move <ID> <column>`) is unchanged.

`move <ID> --to global|project`:

1. Locate the item (search the active store; fall back to searching both stores).
2. **Reallocate** the id from the target store's prefix counter, creating that
   counter if absent (e.g. project `CTX-003` → global → next global `CTX`).
3. **Links:** keep `epic` / `parent` if the referent exists in the target store;
   otherwise **detach** the link (clear the field) and warn.
4. **Epic moves** offer to bring child stories (and their issues) along, each
   reallocated. Declined → the epic moves and orphaned children are detached.
5. Write the item into the target store, remove it from the source, bump
   `updated`, and regenerate **both** stores' `board.md`.
6. Confirm with the remap and any detachments:
   `📁 CTX-003 → 🌐 CTX-005 (epic link detached: not in global)`.

## E. Command impact

| File | Change |
|------|--------|
| `SKILL.md` | §1 resolution rewrite; §2 `scope` field; §6 merged-render rules; §7 cross-store safety |
| `board.md` / `backlog.md` / `story.md` | add `--all` merged read + scope badges |
| `move.md` | add `--to <scope>` cross-store mode |
| `init.md` | write `scope`; accept `--project` |
| `addstory` / `addepic` / `addissue` / `groom` / `pick` / `import` | inherit resolution free; document `--global` / `--project` in arg-hints |
| `README.md` | new "Global vs project stores" section: resolution + walk-up, `--global` / `--project` / `--all`, scope badges, cross-store `move --to`; fix single-store-only examples |

## F. Verification (no test harness — prompt plugin)

Scripted manual scenarios, run and asserted by hand:

- **A** — walk-up from a subdirectory of a project resolves the project store.
- **B** — `--global` from inside a project targets the global store.
- **C** — `board --all` shows both stores badged, and leaves both `board.md`
  files single-scope.
- **D** — cross-store move reallocates the id, detaches + warns on a dangling
  epic link, and regenerates both boards.
- **E** — README documents every new flag (`--global`, `--project`, `--all`,
  `move --to`) and the walk-up + global-fallback behavior, with at least one
  example per feature and no stale single-store-only claims.

## Sequencing

Two commits: **(1)** resolution + scope + merged view + README (§A–C, E-reads);
**(2)** cross-store move (§D).

## Explicitly out of scope (YAGNI)

- `KANBAN_SCOPE` env override — flags + walk-up cover it.
- Merged `board.md` artifact — `--all` is print-only.
