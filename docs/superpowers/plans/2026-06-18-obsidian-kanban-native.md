# cc-kanban v2 — Obsidian Kanban-native Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the cc-kanban plugin so an Obsidian Kanban board `.md` file is the sole source of truth, seed a poolproof board from the bigbrain wiki, and delete the legacy file-store backlog.

**Architecture:** cc-kanban becomes a thin agent-side interface over [Obsidian Kanban](https://github.com/mgmeyers/obsidian-kanban)-format board files. No `.claude/.kanban/` store, no per-item files, no generated `board.md`, no sync layer. All state (columns, ids, settings) is read from the board file itself. The `/kanban:*` slash commands are rewritten as prompt templates that perform minimal, line-scoped edits to a board file; a single `kanban-conventions` skill holds the shared contract.

**Tech Stack:** Markdown prompt files (Claude Code plugin — zero runtime, zero dependencies). Board files conform to the Obsidian Kanban markdown serialization. Verification is dogfooding (the agent follows a command's own prompt to produce an artifact) plus visual conformance in Obsidian.

## Global Constraints

- **Plugin repo (canonical):** `~/workspaces/personal/cc-kanban` — implement here, on branch `feat/obsidian-kanban-native` (already created; spec already committed).
- **Installed copy:** `~/.claude/plugins/marketplaces/cc-kanban/` is a *separate clone*; sync it only in the final task.
- **Board location:** `~/bigbrain/boards/<name>.md` (inside the Obsidian vault). The seeded board is committed in the **bigbrain** repo, not the plugin repo.
- **Zero runtime:** do NOT add scripts, build steps, or dependencies. Commands are prompt templates only.
- **Command namespace:** keep `/kanban:*` (no renames). Plugin version → **2.0.0**.
- **Board = sole truth:** never reintroduce a side store, counter file, or generated board. State lives in the board file (frontmatter + lanes + settings block).
- **Columns:** `Backlog · Todo · Doing · Review · Done` (read from the file's `##` headings; `init` writes these five).
- **Settings block stays last & verbatim:** the `%% kanban:settings %%` block must remain the final element of every board; never reorder or reformat it.
- **Safety:** never delete a card unless explicitly asked; preserve frontmatter and the settings block on every write; a missing/malformed board offers `init`.

---

## Canonical board format (referenced by every task)

`init` writes exactly this skeleton (5 lanes, prefix in frontmatter, settings block last):

````markdown
---
kanban-plugin: board
cc-kanban-prefix: PP
---

## Backlog

## Todo

## Doing

## Review

## Done

%% kanban:settings
```
{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}
```
%%
````

**Card grammar** (the editing contract — same regex everywhere):

```
^(?<indent>\s*)- \[(?<done>[ xX])\] (?:\[(?<id>[A-Z]{2,4}-\d{3,})\] )?(?<text>.*)$
```

- **Story card:** no indent, id token present: `- [ ] [PP-004] Title #epic/slug #p/high #sz/M`
- **Sub-issue:** indented ≥ 2 spaces, no id token, belongs to the nearest preceding story card: `  - [ ] wire webhook`
- **Tags parsed from text:** `#epic/<slug>`, `#p/high|med|low`, `#sz/S|M|L`, `#blocked`, `#paused`.
- **Column = region** between a `## Heading` and the next `##` (or the settings block).
- **ID derivation:** read `cc-kanban-prefix` from frontmatter; new id = (max `PREFIX-NNN` on the board) + 1, else 1; zero-pad to 3. Never reuse within a single operation.

---

## Task 1: Rewrite the `kanban-conventions` skill (the contract)

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/skills/kanban-conventions/SKILL.md`

**Interfaces:**
- Produces (relied on by every command task): the canonical board format, card grammar, board resolution rule, ID derivation rule, column-reading rule, settings-block preservation rule, and safety rules. Command files defer here.

- [ ] **Step 1: Replace the file's body** with the v2 contract. Keep the frontmatter `name`/`description` but update the description to drop "store/columns/id counters" and describe the Obsidian-native model.

Write this content (verbatim sections):

```markdown
---
name: kanban-conventions
description: Core conventions for the kanban plugin — Obsidian Kanban board resolution, card grammar, id derivation, column reading, and safe edits. Every /kanban:* command applies these.
---

# Kanban Conventions (Obsidian Kanban-native)

Shared rules for all `/kanban:*` commands. The **Obsidian Kanban board `.md`
file is the sole source of truth** — there is no separate store, no per-item
files, no generated board. State lives in the board file.

## 1. Resolve the board
- `--board <name|path>`: use that board. A bare name resolves to
  `~/bigbrain/boards/<name>.md`.
- Else: if exactly one `*.md` board exists in `~/bigbrain/boards/`, use it.
- Else (none or many): list candidates and ask which, or offer `init`.
- A board is a `*.md` file whose YAML frontmatter contains `kanban-plugin: board`.

## 2. Board file anatomy
```
---
kanban-plugin: board
cc-kanban-prefix: PP        # id prefix for this board
---

## Backlog
- [ ] [PP-001] Story title #epic/slug #p/high #sz/M
  - [ ] a sub-issue

## Todo
## Doing
## Review
## Done

%% kanban:settings
` ``
{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}
` ``
%%
```
- **Columns** are the `##` headings, in file order. Read them from the file —
  never assume a fixed set beyond what `init` writes.
- The `%% kanban:settings %%` block is **always last**; preserve it verbatim on
  every write.

## 3. Card grammar
`^(\s*)- \[( |x|X)\] (?:\[([A-Z]{2,4}-\d{3,})\] )?(.*)$`
- **Story card:** no indent, has an `[ID]` token.
- **Sub-issue:** indented ≥ 2 spaces, no `[ID]`, belongs to the nearest story above.
- **Tags in text:** `#epic/<slug>` · `#p/high|med|low` · `#sz/S|M|L` · `#blocked` · `#paused`.
- **Done** cards live under `## Done` and are checked `- [x]`.

## 4. ID derivation (stateless)
- Prefix = frontmatter `cc-kanban-prefix`.
- New id = max existing `PREFIX-NNN` on the board + 1 (else 1), zero-padded to 3.
- Never reuse an id within one operation. (A deleted id may later recur — this is
  accepted; we keep no high-water mark.)

## 5. Editing rules
- Edits are **line-scoped and minimal**: insert/move/retag only the affected
  line(s). Never reflow or reformat the whole file.
- Preserve YAML frontmatter and the settings block on every write.
- `move <ID> <column>` relocates the card line (and its indented sub-issues)
  under the target `## Column`; set `- [x]` iff the target is `Done`, else `- [ ]`.

## 6. Read/render
- `board` parses the file and prints an ASCII summary grouped by column, with a
  legend: priority `◆ high · ▣ med · ○ low`, `⛔ blocked · ⏸ paused`. It writes
  nothing.

## 7. Safety
- `init` never clobbers an existing board.
- `import` is idempotent: skip a source that maps to an `[ID]` already present.
- Never delete a card unless explicitly asked.
- A missing/malformed board → offer `init` instead of erroring hard.
```

(Note: in the real file, the three-backtick fences inside the example use real backticks; the ``` ` `` ``` spacing above is only to nest the example here.)

- [ ] **Step 2: Verify the contract is self-consistent**

Run: `grep -nE 'cc-kanban-prefix|kanban-plugin: board|PREFIX-NNN|kanban:settings' ~/workspaces/personal/cc-kanban/skills/kanban-conventions/SKILL.md`
Expected: matches for the prefix key, the board marker, the id rule, and the settings block — confirming all five contract pieces are present.

- [ ] **Step 3: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add skills/kanban-conventions/SKILL.md
git commit -m "feat!: rewrite kanban-conventions for Obsidian Kanban board-as-truth"
```

---

## Task 2: `init` command + version bump

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/init.md`
- Modify: `~/workspaces/personal/cc-kanban/plugin.json` (version `1.1.0` → `2.0.0`, description)
- Modify: `~/workspaces/personal/cc-kanban/.claude-plugin/marketplace.json` (description)

**Interfaces:**
- Consumes: the canonical board format and board-resolution rule from Task 1.
- Produces: a conformant `~/bigbrain/boards/<name>.md` that other commands operate on.

- [ ] **Step 1: Rewrite `commands/init.md`** to this content:

```markdown
---
description: Scaffold an Obsidian Kanban board file (boards/<name>.md).
---

Apply the `kanban-conventions` skill. Create an Obsidian Kanban board.

Args: `<name>` (board base name; default `board`), `--prefix XYZ` (id prefix;
default = uppercase of the first two alphanumerics of `<name>`).

Steps:
1. Resolve target path `~/bigbrain/boards/<name>.md`. If it already exists, STOP
   and report the path (never clobber).
2. Ensure `~/bigbrain/boards/` exists.
3. Write the board skeleton exactly:
   - frontmatter `kanban-plugin: board` + `cc-kanban-prefix: <PREFIX>`
   - the five headings `## Backlog`, `## Todo`, `## Doing`, `## Review`, `## Done`
   - the `%% kanban:settings %%` block with
     `{"kanban-plugin":"board","list-collapse":[false,false,false,false,false]}`
     as the **last** element.
4. Confirm: print the created path and tell the user to open it in Obsidian.
```

- [ ] **Step 2: Bump `plugin.json`**

Set `"version": "2.0.0"` and replace the description with:
`"Obsidian Kanban-native backlog for Claude Code — /kanban:* commands maintain Obsidian board files (boards/*.md) as the single source of truth."`

- [ ] **Step 3: Update `marketplace.json` description** to match the new one-liner above.

- [ ] **Step 4: Dogfood-verify init produces a conformant board**

Follow the rewritten `init.md` to create a throwaway board:
Run: create `~/bigbrain/boards/_smoke.md` per the init steps, then
`cat ~/bigbrain/boards/_smoke.md`
Expected: frontmatter has `kanban-plugin: board` and `cc-kanban-prefix: SM`; five `##` headings in order; settings block is the final element.

- [ ] **Step 5: Verify it renders in Obsidian (manual)**

Open `_smoke.md` in Obsidian. Expected: it displays as a Kanban board with five empty lanes. Then delete the smoke file:
Run: `rm ~/bigbrain/boards/_smoke.md`

- [ ] **Step 6: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add commands/init.md plugin.json .claude-plugin/marketplace.json
git commit -m "feat!: init writes Obsidian Kanban board; bump to 2.0.0"
```

---

## Task 3: Create commands — `addstory`, `addissue`, `addepic`

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/addstory.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/addissue.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/addepic.md`

**Interfaces:**
- Consumes: card grammar + ID derivation (Task 1); a board created by `init` (Task 2).
- Produces: card lines other commands read/move.

- [ ] **Step 1: Rewrite `commands/addstory.md`:**

```markdown
---
description: Add a story card to a board column (default Backlog).
---

Apply the `kanban-conventions` skill. Append a story card to a board.

Args: `<desc>` (required), `--epic <slug>`, `--priority high|med|low`,
`--size S|M|L`, `--col <Column>` (default `Backlog`), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1). If none, offer `init` and STOP.
2. Derive the new id (skill §4): prefix from frontmatter, max+1.
3. Build the card text:
   `- [ ] [<ID>] <desc>` then append, when provided:
   ` #epic/<slug>` ` #p/<priority>` ` #sz/<size>`.
4. Insert the line as the LAST card under the `## <Column>` heading (just before
   the next `##` or the settings block). Preserve everything else.
5. Confirm: print the new id, column, and tags.
```

- [ ] **Step 2: Rewrite `commands/addissue.md`:**

```markdown
---
description: Add a sub-issue (nested checkbox) under a story card.
---

Apply the `kanban-conventions` skill. Add a sub-issue to a story.

Args: `<desc>` (required), `--story <ID>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Find the story card line whose id token is `[<ID>]`. If not found, STOP and
   report.
3. Insert `  - [ ] <desc>` (two-space indent) immediately after the story card's
   existing sub-issues (or right after the card line if it has none).
4. Confirm: print the parent id and the sub-issue text.
```

- [ ] **Step 3: Rewrite `commands/addepic.md`:**

```markdown
---
description: Declare an epic tag for use on cards (epics are #epic/<slug> tags).
---

Apply the `kanban-conventions` skill. Epics are not files or columns — they are
`#epic/<slug>` tags applied to story cards.

Args: `<slug>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Normalize `<slug>` to kebab-case.
3. Report the tag to use: ``apply `#epic/<slug>` to cards via `addstory --epic
   <slug>` or `groom``. Mention `backlog --epic <slug>` filters by it.
   (No file is created; the epic "exists" once a card carries the tag.)
```

- [ ] **Step 4: Dogfood-verify the create path**

Create a scratch board, then follow `addstory`, `addissue`:
Run (follow the prompts): `init _smoke --prefix SM`; `addstory "First story" --epic demo --priority high --size M --board _smoke`; `addissue "a subtask" --story SM-001 --board _smoke`; then `cat ~/bigbrain/boards/_smoke.md`
Expected: under `## Backlog`:
```
- [ ] [SM-001] First story #epic/demo #p/high #sz/M
  - [ ] a subtask
```
and the settings block is still last.

- [ ] **Step 5: Clean up scratch board**

Run: `rm ~/bigbrain/boards/_smoke.md`

- [ ] **Step 6: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add commands/addstory.md commands/addissue.md commands/addepic.md
git commit -m "feat: addstory/addissue/addepic operate on board cards"
```

---

## Task 4: Mutate commands — `move`, `groom`

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/move.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/groom.md`

**Interfaces:**
- Consumes: card grammar, column-as-region, Done-toggle rule (Task 1).
- Produces: cards relocated/retagged in place.

- [ ] **Step 1: Rewrite `commands/move.md`** (drop the v1 `--to global|project` cross-store mode entirely — single vault now):

```markdown
---
description: Move a card to another column (status change).
---

Apply the `kanban-conventions` skill. Relocate a card between columns.

Args: `<ID>` (required), `<column>` (required; must match an existing `##`
heading, case-insensitive), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Read the `##` headings; validate `<column>` matches one. If not, list valid
   columns and STOP.
3. Find the story card line `[<ID>]` and capture its indented sub-issues (the
   contiguous deeper-indented `- [ ]` lines beneath it).
4. Remove the card + its sub-issues from their current column.
5. Append them as the last card under the target `## <column>`.
6. Set the card's checkbox: `- [x]` iff target is `Done`, else `- [ ]`.
   (Leave sub-issue checkboxes unchanged.)
7. Preserve frontmatter and the settings block. Confirm: `<ID> → <column>`.
```

- [ ] **Step 2: Rewrite `commands/groom.md`:**

```markdown
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
```

- [ ] **Step 3: Dogfood-verify move toggles Done**

Run (follow prompts): `init _smoke --prefix SM`; `addstory "Movable" --board _smoke`; `move SM-001 Done --board _smoke`; `cat ~/bigbrain/boards/_smoke.md`
Expected: `- [x] [SM-001] Movable` appears under `## Done` and is gone from `## Backlog`.

- [ ] **Step 4: Verify leaving Done unchecks**

Run (follow prompts): `move SM-001 Todo --board _smoke`; `grep -n 'SM-001' ~/bigbrain/boards/_smoke.md`
Expected: `- [ ] [SM-001] Movable` under `## Todo` (checkbox cleared). Then `rm ~/bigbrain/boards/_smoke.md`.

- [ ] **Step 5: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add commands/move.md commands/groom.md
git commit -m "feat!: move toggles Done in-file; drop cross-store mode; groom retags cards"
```

---

## Task 5: Read commands — `board`, `backlog`, `story`, `pick`

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/board.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/backlog.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/story.md`
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/pick.md`

**Interfaces:**
- Consumes: card grammar, columns-from-headings, tag parsing (Task 1). Read-only — none write to the board.

- [ ] **Step 1: Rewrite `commands/board.md`** (no longer generates a file):

```markdown
---
description: Print an ASCII summary of a board (read-only). --open opens it in Obsidian.
---

Apply the `kanban-conventions` skill. Render a board to the terminal.

Args: `--epic <slug>` (filter), `--open`, `--board <name|path>`.

Steps:
1. Resolve the board (skill §1). If `--open`, open it via `obsidian://open?path=`
   (URL-encode the vault-relative path) and stop.
2. Parse columns (`##` headings) and their cards. Optionally filter to
   `#epic/<slug>`.
3. Print one section per column; each card on a line as
   `<glyph> [ID] text` using priority glyphs `◆ high · ▣ med · ○ low`
   (and `⛔` if `#blocked`, `⏸` if `#paused`). Show sub-issue counts as `(n/m)`.
4. Print a legend. Write NOTHING to disk.
```

- [ ] **Step 2: Rewrite `commands/backlog.md`:**

```markdown
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
```

- [ ] **Step 3: Rewrite `commands/story.md`:**

```markdown
---
description: Show one card with its sub-issues and tags.
---

Apply the `kanban-conventions` skill. Show a single card.

Args: `<ID>` (required), `--board <name|path>`.

Steps:
1. Resolve the board (skill §1).
2. Find `[<ID>]`. Print its column, full text, parsed tags
   (epic/priority/size/flags), and its sub-issues with checkbox state.
3. If not found, report and suggest `backlog`.
```

- [ ] **Step 4: Rewrite `commands/pick.md`:**

```markdown
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
```

- [ ] **Step 5: Dogfood-verify read commands**

Run (follow prompts): `init _smoke --prefix SM`; `addstory "Low task" --priority low --board _smoke`; `addstory "High task" --priority high --board _smoke`; then follow `pick --board _smoke`.
Expected: `pick` recommends `SM-002 High task` (high beats low). Then follow `board --board _smoke` and confirm two cards print under Backlog with glyphs and a legend, and nothing was written (file mtime unchanged aside from the addstory edits). Then `rm ~/bigbrain/boards/_smoke.md`.

- [ ] **Step 6: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add commands/board.md commands/backlog.md commands/story.md commands/pick.md
git commit -m "feat!: read commands parse board file; board no longer generates board.md"
```

---

## Task 6: `import` command (migration engine)

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/commands/import.md`

**Interfaces:**
- Consumes: board format, ID derivation, idempotency rule (Task 1); `init` (Task 2); `addstory` insertion behavior (Task 3).
- Produces: cards on a board derived from external markdown — used by Task 7 to seed poolproof.

- [ ] **Step 1: Rewrite `commands/import.md`:**

```markdown
---
description: Adopt existing markdown into board cards (idempotent).
---

Apply the `kanban-conventions` skill. Import markdown into a board.

Args: `<source>` (a file or directory of markdown), `--board <name|path>`
(create with `init` if absent), `--epic <slug>` (default epic for imported
cards), `--col <Column>` (default `Backlog`).

Steps:
1. Resolve or `init` the board.
2. Read `<source>`:
   - A directory of per-item files: each file → one story card (title from the
     first `#` heading or filename; map any `priority`/`size` frontmatter to
     `#p/`/`#sz/`).
   - A single structured page (e.g. a wiki status/open-questions page): map
     sections to columns where the section name implies status (e.g.
     "Recently done"→Done, "Next up"→Todo, "In progress"→Doing,
     unchecked bullets under "Open questions"→Backlog, checked `[x]`→Done).
3. **Idempotency:** before adding, derive a stable slug per source item; skip any
   item already represented on the board (match by title text). Never duplicate.
4. Assign ids via skill §4 as you add. Apply `--epic`/tags.
5. Insert cards under their mapped columns (or `--col`). Preserve the settings
   block.
6. Report: counts added per column, and anything skipped.
```

- [ ] **Step 2: Dogfood-verify import maps sections to columns**

Create a tiny source file and import it:
Run: write `~/tmp/_src.md` containing:
```
## Recently done
- Shipped thing A
## Next up
- Plan thing B
```
Then follow `import ~/tmp/_src.md --board _smoke --epic demo` (init `_smoke` first if needed), then `cat ~/bigbrain/boards/_smoke.md`.
Expected: "Shipped thing A" is a `- [x]` card under `## Done`; "Plan thing B" is a `- [ ]` card under `## Todo`; both tagged `#epic/demo`.

- [ ] **Step 3: Verify idempotency**

Run: follow `import ~/tmp/_src.md --board _smoke --epic demo` a second time, then `grep -c 'thing A' ~/bigbrain/boards/_smoke.md`
Expected: `1` (no duplicate). Then `rm ~/bigbrain/boards/_smoke.md ~/tmp/_src.md`.

- [ ] **Step 4: Commit**

```bash
cd ~/workspaces/personal/cc-kanban
git add commands/import.md
git commit -m "feat!: import adopts markdown into board cards (section→column mapping, idempotent)"
```

---

## Task 7: Seed `boards/poolproof.md` from the wiki

**Files:**
- Create: `~/bigbrain/boards/poolproof.md` (committed in the **bigbrain** repo)

**Interfaces:**
- Consumes: `init` (Task 2) + `import` logic (Task 6).
- Produces: the first real board.

- [ ] **Step 1: Create the board**

Follow `init poolproof --prefix PP` to create `~/bigbrain/boards/poolproof.md`.

- [ ] **Step 2: Seed cards from the poolproof wiki**

Read `~/bigbrain/wiki/projects/poolproof/status.md` and
`~/bigbrain/wiki/projects/poolproof/open-questions.md` and add cards via the
mapping below (epics: `#epic/suspend-flow` for shipped work, `#epic/route-reconcile`
for follow-ups). Assign ids `PP-001…` in this order:

- **Done** (`- [x]`):
  - `[PP-001] Suspend → route-map refresh client fix (fix-edit-wo-dropdowns) #epic/suspend-flow #p/high #sz/S`
  - `[PP-002] Day-of-hold server reconcile (7af90770) #epic/suspend-flow #p/high #sz/M`
  - `[PP-003] Verified full server reconcile end-to-end (route 3478) #epic/suspend-flow #p/med #sz/S`
- **Todo**:
  - `[PP-004] Decide on resume-reconcile approach + hook point #epic/route-reconcile #p/high #sz/S`
- **Backlog**:
  - `[PP-005] Resume-reconcile: recompute route when a hold is lifted #epic/route-reconcile #p/med #sz/M`
  - `[PP-006] Confirm reconcile covers already-generated future-week routes #epic/route-reconcile #p/med #sz/S`
  - `[PP-007] End-bounded suspensions: handle any path that sets suspendedUntil #epic/route-reconcile #p/low #sz/S`
  - `[PP-008] Day-of-hold billing (paused — in git stash; leave alone) #epic/suspend-flow #p/low #sz/M #paused`

Leave `## Doing` and `## Review` empty.

- [ ] **Step 3: Verify the board parses and renders**

Run: `cat ~/bigbrain/boards/poolproof.md`
Expected: frontmatter `kanban-plugin: board` + `cc-kanban-prefix: PP`; 8 cards across Done/Todo/Backlog; settings block last.
Then open it in Obsidian. Expected: renders as a Kanban board; `PP-008` shows the `#paused` tag; Done cards are checked.

- [ ] **Step 4: Verify with the plugin's own read command**

Follow `board --board poolproof`. Expected: ASCII summary shows 3 Done, 1 Todo, 4 Backlog, with priority glyphs and `⏸` on PP-008.

- [ ] **Step 5: Commit (bigbrain repo)**

```bash
cd ~/bigbrain
git add boards/poolproof.md
git commit -m "feat: seed Obsidian Kanban poolproof board from wiki"
```

---

## Task 8: Delete the legacy data

**Files:**
- Archive then delete: `~/.claude/stories/` (preserve the original plugin design doc)
- Delete: `~/.claude/.kanban/`
- Create (archive): `~/workspaces/personal/cc-kanban/docs/superpowers/specs/2026-06-16-kanban-plugin-design.md`

**Interfaces:** none (filesystem cleanup). Do this only after Task 7's board is confirmed.

- [ ] **Step 1: Locate and archive the original design doc**

Run: `fd -H -i 'kanban-plugin-design' ~/.claude/stories`
Expected: a path like `~/.claude/stories/specs/2026-06-16-kanban-plugin-design.md`.
Copy it into the plugin repo:
Run: `cp "<found path>" ~/workspaces/personal/cc-kanban/docs/superpowers/specs/2026-06-16-kanban-plugin-design.md`
(If no such file exists, skip the archive and note it.)

- [ ] **Step 2: Confirm deletion targets before removing**

Run: `ls -d ~/.claude/.kanban ~/.claude/stories 2>/dev/null` and
`ls -d ~/workspaces/personal/claude-stats/.claude/.kanban 2>/dev/null`
Expected: the first two exist (to delete); the claude-stats store is listed only to confirm we will NOT touch it.

- [ ] **Step 3: Delete the legacy global store and stories**

Run: `rm -rf ~/.claude/.kanban ~/.claude/stories`
Then verify: `ls -d ~/.claude/.kanban ~/.claude/stories 2>/dev/null || echo "removed"`
Expected: `removed`. Confirm claude-stats untouched: `ls ~/workspaces/personal/claude-stats/.claude/.kanban/` still lists its files.

- [ ] **Step 4: Commit the archived design doc**

```bash
cd ~/workspaces/personal/cc-kanban
git add docs/superpowers/specs/2026-06-16-kanban-plugin-design.md
git commit -m "docs: archive original v1 kanban-plugin design before deleting legacy store"
```

---

## Task 9: README rewrite + sync installed copy + PR

**Files:**
- Modify (full rewrite): `~/workspaces/personal/cc-kanban/README.md`

**Interfaces:** none new. Final integration + publish.

- [ ] **Step 1: Rewrite `README.md`** to describe the v2 model: Obsidian Kanban board files in `~/bigbrain/boards/` as the single source of truth, the card grammar (`[ID]`, `#epic/ #p/ #sz/`), the `/kanban:*` command table (init, addstory, addissue, addepic, move, board, backlog, story, groom, pick, import), and that there is no store/`board.md`/sync. Remove all references to `.claude/.kanban/`, `config.json`, global-vs-project stores, `--all` merged views, and cross-store `move --to`.

- [ ] **Step 2: Commit the README**

```bash
cd ~/workspaces/personal/cc-kanban
git add README.md
git commit -m "docs: rewrite README for Obsidian Kanban-native v2"
```

- [ ] **Step 3: Push and open the PR**

```bash
cd ~/workspaces/personal/cc-kanban
git push -u origin feat/obsidian-kanban-native
gh pr create --fill --title "cc-kanban v2 — Obsidian Kanban-native"
```

- [ ] **Step 4: Sync the installed marketplace copy so `/kanban:*` reflects v2**

After the PR merges (or to test locally), update the installed clone:
Run: `git -C ~/.claude/plugins/marketplaces/cc-kanban pull` (or, from Claude Code, `/plugin marketplace update cc-kanban`).
Then verify: `grep '"version"' ~/.claude/plugins/marketplaces/cc-kanban/plugin.json`
Expected: `2.0.0`. Restart the session so the updated commands load.

- [ ] **Step 5: Final end-to-end smoke**

In a fresh session, follow `board --board poolproof`. Expected: the v2 summary prints (3 Done / 1 Todo / 4 Backlog). Open `~/bigbrain/boards/poolproof.md` in Obsidian and drag PP-004 to Doing; re-run `story PP-004` and confirm it reports column `Doing` — proving the round-trip (Obsidian writes, cc-kanban reads the same file).

---

## Self-Review

**Spec coverage:**
- Data model (board = sole truth) → Tasks 1, 2, and the Global Constraints. ✓
- Board format + card conventions → Task 1 (contract) + canonical format block. ✓
- Columns Backlog…Done → Task 2 (init writes them), read from headings. ✓
- Full command surface (init, addstory, addissue, addepic, move, board, backlog, story, groom, pick, import) → Tasks 2–6. ✓
- Migration (poolproof from wiki) → Task 7 (uses Task 6 import logic). ✓
- Cleanup (delete store + stories, archive design doc, leave claude-stats) → Task 8. ✓
- Implementation target + installed-copy sync + version 2.0.0 → Tasks 2, 9 + Global Constraints. ✓
- Out-of-scope removals (cross-store move, `--all`, per-item store) → Tasks 4, 5, 9 explicitly remove them. ✓

**Placeholder scan:** command bodies, the conventions contract, the canonical board template, the card grammar regex, and the seed card list are all concrete. No TBD/TODO. ✓

**Type/name consistency:** `cc-kanban-prefix` (frontmatter), `[PREFIX-NNN]` id token, `#epic/ #p/ #sz/ #blocked #paused` tags, the five column headings, and the settings-block JSON are used identically across Tasks 1–7. `board` is read-only in every task that mentions it. ✓
```
