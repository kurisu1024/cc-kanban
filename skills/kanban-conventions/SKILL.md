---
name: kanban-conventions
description: Core conventions for the kanban plugin — store resolution, item schema, id allocation, status columns, and board rendering. Every /kanban:* command applies these so they behave consistently.
---

# Kanban Conventions

Shared rules for all `/kanban:*` commands. When a kanban command runs, apply
these rules. They are the single source of truth — command files stay thin and
defer here.

## 1. Resolve the store directory

There are two stores: a **project store** at some `<repo>/.claude/.kanban/` and
the **global store** at `~/.claude/.kanban/`. Most commands act on a single
**active store** (the read/write target), resolved in this order:

1. `--global` → the global store (`~/.claude/.kanban/`).
2. `--project` → the nearest project store via walk-up (below). If none is
   found, stop and offer `init` — do **not** silently fall back to global.
3. **Default:** walk up from cwd for a project store; first hit wins. If none,
   use the global store.

**Walk-up rule:** starting at cwd, check each ancestor directory `D` for
`D/.claude/.kanban/`; the first match is the project store. Stop at the
filesystem root. **Exclude `$HOME` itself** from the walk — `~/.claude/.kanban/`
*is* the global store and must never be matched as a project store. So the
global store is reached only via `--global` or as the default fallback.

- If a writing command resolves to a store that does not exist yet, scaffold it
  first (see `init`).
- `--all` is not an active-store selector; it is a **read-only merged view**
  across both stores (see §6).

Layout:

```
.claude/.kanban/
  config.json        # scope, columns, id counters, settings
  board.md           # GENERATED view — never the source of truth
  epics/   stories/  issues/
```

## 2. config.json

```json
{
  "scope": "project",
  "columns": ["backlog", "todo", "doing", "review", "done"],
  "idCounters": { "EPIC": 0, "ISS": 0 },
  "settings": { "defaultStatus": "backlog", "groupBoardByEpic": true }
}
```

- `scope` is `"project"` or `"global"`, written at `init` (`"global"` under
  `--global`). It labels the store in merged views and tells cross-store moves
  which side they are on.
- Per-epic story prefixes (e.g. `CTX`) are added to `idCounters` as epics are
  created or imported.
- `idCounters` are **per-store and independent**. Global `CTX-001` and project
  `CTX-001` may both exist; they only ever appear side by side in the merged
  view, where the scope badge (§6) disambiguates them.

## 3. Item files

- Filename: `<ID>-<kebab-title>.md` in the subdir matching its `type`.
- Frontmatter:

```yaml
id:        # epic → EPIC-NNN · story → <PREFIX>-NNN · issue → ISS-NNN
type:      # epic | story | issue
epic:      # story → parent epic's prefix (epics: a `prefix:` field instead)
parent:    # issue → parent story id
status:    # one of config.columns
priority:  # high | medium | low
size:      # S | M | L
labels: []
created:   # YYYY-MM-DD (today)
updated:   # YYYY-MM-DD (today)
```

- Body: free markdown (problem / solution / acceptance criteria / notes).
- Use today's date for `created`/`updated`.

## 4. ID allocation (deterministic)

- Read the relevant counter in `config.json.idCounters`, increment it, write it
  back, then use the new value. **Never reuse an id.**
- Epics: `EPIC-NNN` (zero-padded to 3). Each epic also declares a story `prefix`.
- Stories: `<EPIC_PREFIX>-NNN` from that epic's counter. With no epic, use a
  default `STORY` prefix/counter.
- Issues: `ISS-NNN`.

## 5. Status / columns

- Move items only between configured `columns`; validate the target.
- On move: set `status`, bump `updated` to today, then regenerate the board.

## 6. Board (`board.md` + printed)

- Group by column; within a column, group by epic when
  `settings.groupBoardByEpic`.
- Print an ASCII board to the user **and** write the same content as markdown
  tables to `board.md`.
- Priority glyphs: `◆` high · `▣` medium · `○` low (include a legend).

### Merged view (`--all`)

When a read command (`board`, `backlog`, `story`) is given `--all`, union the
items from **both** stores (project + global; if only one exists, `--all` equals
that one):

- Badge every item by its store: `🌐` global · `📁` project. Add these to the
  legend alongside the priority glyphs.
- Layout: column → scope subsection → epic grouping. The badge disambiguates ids
  that exist in both stores.
- `--all` is **print-only**. Never write a merged board to either `board.md`;
  each store's `board.md` stays single-scope and canonical. (Regenerate a store's
  own `board.md` only from that store's own items.)

## 7. Idempotency & safety

- `init` never clobbers an existing store.
- `import` skips ids that already exist; only fills in missing `status`/`epic`.
- Always regenerate `board.md` after any mutation.
- `board.md` is derived — never treat hand-edits to it as truth.
- Never delete item files unless explicitly asked.
- Keep frontmatter valid YAML; keep writes minimal and scoped.

## 8. Cross-store moves (promote / demote)

`move <ID> --to global|project` relocates an item between stores (distinct from
`move <ID> <column>`, which only changes `status`). Steps:

1. Locate the item: search the active store; if not found there, search the
   other store. Error if the id is unknown or already lives in the target scope.
2. **Reallocate the id** from the *target* store's counter for that id family
   (the epic prefix for stories, or `EPIC` / `ISS`), creating the counter if
   absent. Never carry the source id across — that would risk a collision.
3. **Links:** keep `epic` (story) or `parent` (issue) only if the referent
   exists in the target store; otherwise **detach** it (clear the field) and warn
   the user which link was dropped.
4. **Moving an epic:** offer to bring its child stories (and their issues) along,
   each reallocated by this same procedure. If declined, move the epic alone and
   leave its source-store children detached from it.
5. Write the item into the target store's matching subdir, remove it from the
   source store, and bump `updated`.
6. Regenerate **both** stores' `board.md` (both changed).
7. Confirm with the remap and any detachments, e.g.
   `📁 CTX-003 → 🌐 CTX-005 (epic link detached: "Context & Memory IQ" not in global)`.
