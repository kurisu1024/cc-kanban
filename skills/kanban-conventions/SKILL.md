---
name: kanban-conventions
description: Core conventions for the kanban plugin — store resolution, item schema, id allocation, status columns, and board rendering. Every /kanban:* command applies these so they behave consistently.
---

# Kanban Conventions

Shared rules for all `/kanban:*` commands. When a kanban command runs, apply
these rules. They are the single source of truth — command files stay thin and
defer here.

## 1. Resolve the store directory

- If `<cwd>/.claude/.kanban/` exists → use it (**project store**).
- Else → use `~/.claude/.kanban/` (**global store**).
- A `--global` flag always forces `~/.claude/.kanban/`.
- If a writing command finds no store, scaffold it first (see `init`).

Layout:

```
.claude/.kanban/
  config.json        # columns, id counters, settings
  board.md           # GENERATED view — never the source of truth
  epics/   stories/  issues/
```

## 2. config.json

```json
{
  "columns": ["backlog", "todo", "doing", "review", "done"],
  "idCounters": { "EPIC": 0, "ISS": 0 },
  "settings": { "defaultStatus": "backlog", "groupBoardByEpic": true }
}
```

Per-epic story prefixes (e.g. `CTX`) are added to `idCounters` as epics are
created or imported.

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

## 7. Idempotency & safety

- `init` never clobbers an existing store.
- `import` skips ids that already exist; only fills in missing `status`/`epic`.
- Always regenerate `board.md` after any mutation.
- `board.md` is derived — never treat hand-edits to it as truth.
- Never delete item files unless explicitly asked.
- Keep frontmatter valid YAML; keep writes minimal and scoped.
