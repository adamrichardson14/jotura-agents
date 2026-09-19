---
name: jotura-notes
description: Filing and capture rules for the user's Jotura vault — which folder content belongs in, what a finished note must contain, where inside an existing note new content goes (into its section, never appended to the bottom), and the conventions and memory loops that open and close a task. Invoke this BEFORE creating, moving, or filing anything, and BEFORE proposing an approach, architecture, or plan: it names the vault commands to run first and how to act on what they return. Invoke it proactively at capture checkpoints even when the user says nothing about notes — a decision or design is approved, a plan is written, a bug's root cause is found, a piece of work is finished, substantive research concludes. Never file from memory or from a summary in CLAUDE.md; the rules are vault-specific, learned over time, and live in the vault's own Agent Conventions note. The jotura-vault skill covers the CLI mechanics — use both together.
---

# Jotura notes — filing & capture

This skill ships general principles and no knowledge of this user's vault. That knowledge lives **in the vault itself**: the Agent Conventions note, shared by every AI agent the user runs, synced across their devices, and never touched by installers. `jotura conventions` prints it, creating a seeded template on first use.

## The improvement loop — every session

1. **Before filing anything, run `jotura conventions`** and follow what it says. It is the agents' shared memory of how this vault is organized.
2. **The moment you learn something new — a folder's purpose, a naming convention, a filing correction from the user — update the conventions note in the same session** (normal edit: `jotura edit "Agent Conventions.md" ...`). A correction from the user is the strongest signal there is; it always earns an entry.
3. Never record secrets in the conventions note, and never let it grow stale: if filing ever feels ambiguous, that ambiguity is a missing convention — resolve it with the user and write it down.

This loop is what makes agents get better at this vault over time. Skipping step 2 means the next session repeats today's mistakes.

## The memory loop — learn from the user's decisions

Beyond filing, the vault is the agents' long-term memory of the user's work: their deliberate decisions, facts about their systems and projects, and open follow-ups. Memory notes live under `Memory/` (kinds: `Decisions/`, `Facts/`, `Followups/`) with structured frontmatter, fully visible and correctable by the user. Memory records the work — never personal traits, characterizations, or secrets.

**Read side (what makes advice good):**

- At task start, run `jotura context "<task description>"` — it returns the conventions, ranked relevant memories, and open follow-ups in one pack. Plain `jotura context` gives a session digest.
- **Before proposing an approach, architecture, tool choice, or plan, `jotura memory recall "<topic>"` is mandatory.** Recall defaults to fused semantic plus keyword and degrades to keyword on its own when the vault has no vector index, so it never blocks the loop. Surface what you find: "you decided X on <date> because Y — this proposal aligns / conflicts". Never silently contradict an `active` decision; flag it, and let the resolution become a supersede.
- Treat old memories as claims to confirm, not laws: recall already downranks stale entries, and a decision the user re-affirms is worth re-dating.

**Write side (what makes the next session smarter):**

- At the capture checkpoints (decision approved, plan written, root cause found, work finished, research concluded), also log structured memory: `jotura memory log --kind decision --title "..." [--project X] [--importance high] [--body -]`. Log liberally — importance and recall-ranking do the filtering, not the writer.
- Record when something became true, not just when you logged it: `memory log --occurred YYYY-MM-DD` for anything that happened before today. `date` stays the logging date; staleness and ranking are measured from `occurred`, so a backdated decision is correctly treated as old.
- Established facts about the user's world ("X is the HR system", "deploys go through Y") → `--kind fact`. Anything deferred or worth revisiting → `--kind followup`; resolve it when done (`jotura memory resolve`).
- `memory log` returns `similar:` hits — if one covers the same ground, supersede it (`jotura memory supersede <old> --by <new>`) instead of duplicating.
- When finished work proves a decision out (or not), append the outcome to that decision note.
- **Sweep periodically with `jotura memory doctor`** (add `--fix` to backfill missing `occurred` dates). It reports stale actives, follow-ups left open too long, supersede edges pointing at deleted notes, and memory filed under the bare default root instead of the project's. Act on what it reports: a stale active gets confirmed or superseded, not ignored. `--fix` only makes mechanical repairs; anything needing a judgement call is left for the user.
- The user can browse and correct memory themselves: the desktop app's Agent Memory tab (command palette, "Open Agent Memory") shows the timeline grouped by month with supersede chains inline. Point them there when they question what an agent remembers. Memory the user cannot see is memory they cannot fix.

## First: learn the structure

Before writing anything, run `jotura conventions`, then `jotura ls --json` to study the top-level folders.

- If the vault has an established structure, follow it. Never invent parallel folders for content that already has a home.
- **If the vault is new, empty, or its structure is unclear: stop and ask the user how they organize their notes.** Offer a minimal starter layout as a suggestion — `Daily/` (one note per day), `Projects/` (one folder per project), `Research/`, `Reference/` — and create the agreed structure before filing anything.
- Record what you learn in the conventions note, the same session you learn it.

## Capture proactively — the standing rule

Substantive outcomes get written into the vault as a matter of course, not on request. These checkpoints always trigger a capture:

- a **decision or design is approved** → record the choice and why
- a **plan is written** → record what it builds and where the plan lives
- a **bug's root cause is found** → record the one-line cause and fix
- a **piece of work is finished** → record the outcome (merged, shipped, delivered)
- **substantive research concludes** → record findings and comparisons

File by project: identify the project the work belongs to and write into that project's folder or anchor note. If the project has no folder yet, create one — ask the user first while the vault's conventions are still young.

When the target note already exists, place the content inside it by the rules in the next section. Do not append.

## Adding to an existing note: place it, never append it

Appending to the bottom of a note is the most common filing mistake agents make, and it is almost always wrong. A note is a structured document with sections, not a log. New content goes where a reader would look for it; `--append` puts it below `## Links`, where nobody looks.

The procedure, every time the target note already exists:

1. **Read the whole note first** (`jotura read <path> --numbered`, plus `--json` for the hash). Never add to a note you have not read in this session: the read is what tells you where the content belongs.
2. **Find the home for the content inside the note**: the heading on the same topic, the dated section for today, the list it extends, the table row it updates. If the note already says the thing, revise that sentence in place rather than adding a second version of it.
3. **Insert there**, after the last line of that section (`--insert-after <line>`), or by replacing a unique nearby line with itself plus the new lines (`--replace ... --with`). A new dated entry goes beside its sibling entries in the note's existing order (newest-first or newest-last, whichever the note already uses), not at the end of the file.
4. **No section fits? Create one** in the note's own heading style and level, at the position where it belongs in the note's structure. A `## Update` heading tacked onto the bottom is still an append.
5. **`## Links` stays last.** Merge any new wikilinks into it; never write anything below it.
6. **Re-read the changed region** and confirm the content sits where you intended and the note still reads as one document.

`--append` is only for notes that are append-by-design: the daily note's bullets (`jotura today --append`) and logs whose stated convention is newest-at-bottom. On any other note, reaching for `--append` means the read step was skipped. The vault's Agent Conventions note may name further append-by-design notes or add placement rules; follow it.

## What renders in Jotura

Jotura renders CommonMark and GFM plus the Obsidian syntax below. Every form
here is a first-class editor feature, kept byte-for-byte in the file through
opening, saving, and editing the block it sits in. Write it as plain text; no
escaping is needed.

| Write | Renders as |
|---|---|
| `[[Note]]`, `[[Note\|alias]]` | styled, clickable wikilink; the brackets stay visible and the alias is not substituted |
| `[[Note#Heading]]`, `[[Note#^block]]` | the same link to `Note`; the anchor is kept in the file and ignored when resolving, so the note opens but does not scroll to the heading. `[[#Heading]]` is the note you are in |
| `![[Note]]` | an embed chip carrying the note name; the note's text is not pulled in |
| `![[image.png]]`, `![[image.png\|300]]`, `![[image.png\|300x200]]` | the image inline, at that pixel width (and height); a non-image file embeds as a chip carrying its name |
| `![alt](path.png)` | image |
| `> [!kind] Title` | a callout: a titled block with a kind icon and a clickable kind badge. Thirteen kinds are known (note, abstract, info, todo, tip, success, question, warning, failure, danger, bug, example, quote) and any other word still renders; the warning family (warning, caution, danger, bug, error, failure) is tinted. Callouts nest |
| `> [!kind]-` / `> [!kind]+` | the same, collapsed / expanded by default; the fold state is written into the file |
| `#tag`, `#area/sub` | clickable tag pill; must start a line or follow a space, first character a letter, digit or underscore. Counted in the Tags pane alongside frontmatter `tags`, except inside code |
| `- [ ] text @due(2026-09-30)` | a task in the Tasks pane, dated by `@due`, or by the daily note's date when written in a daily note; `- [x]` marks it done |
| `- [/]`, `- [-]`, `- [>]`, `- [?]` | in-progress, cancelled, forwarded and question markers, rendered with their own checkbox styling. Only `[ ]`, `[x]` and `[X]` count as tasks in the Tasks pane, so date-tracked work needs one of those three |
| `==highlight==` | a highlight mark |
| `$\pi r^2$`, `$$ ... $$` | inline and block math, shown as monospace text; nothing is typeset |
| `%% comment %%` | a muted aside, kept in the file and dropped from HTML export; `%%` alone on its own lines makes a whole block a comment |
| `[^1]` and `[^1]: text` | a footnote reference and its definition, shown as written with light styling on the marker. Labels may be words, with no spaces or brackets |
| `^block-id` at the end of a line | a block id, shown as written; `[[Note#^block-id]]` links to the note holding it |
| tables, task lists, `~~strike~~`, fenced code, blockquotes, `---` rules | as GFM |
| ```` ```mermaid ```` fence | rendered diagram |
| `---` YAML frontmatter | hidden in the editor, editable in source mode |

**Do not write:** inline HTML. The Markdown layer has HTML disabled, so a
snippet of markup is shown as ordinary text. An `.excalidraw.md` note opens in
source mode and its drawing is not rendered.

You can look at the images a note embeds: `jotura links <note> --json` gives
each embed's `absolutePath`, and your own image-capable file tool opens it.
Read the jotura-vault skill's `references/images.md` before viewing or adding
images.

**Frontmatter contract.** The app reads two keys. `tags` (inline `[a, b]`, a
`- item` list, or a bare scalar) fills the Tags pane, together with the body's
inline `#tags`. `aliases` (the same three shapes) gives the note extra names
that wikilinks, quick open, and the link picker all resolve, which is how a
renamed note keeps its old links working. Every other key is preserved verbatim
and ignored: `title` does not rename a note, titles come from the filename.
The top-level `jotura` key is reserved for Jotura's per-note presentation
attributes (today table column widths), written as one JSON line; leave it in
place when editing other frontmatter and never add keys under it by hand.
Memory notes under `Memory/` carry `kind`, `status`, `project`, `date`,
`importance` and `supersedes`, read only by `jotura memory`. Write dates as
`YYYY-MM-DD`. Set keys with `jotura frontmatter set` and tags with
`jotura tag add`; `edit` never touches the block. `jotura tags` counts
frontmatter and inline `#tags` like the Tags pane, while `jotura tag list`
reports only the frontmatter block it edits.

## What belongs in notes — and what doesn't

**DO capture:**

- decisions with their rationale — the why matters more than the what
- plans and specs as a pointer: what it builds and where the document lives
- debugging root causes and troubleshooting conclusions
- research findings, tool comparisons, meeting outcomes
- anything the user would want to find again in six months

**Do NOT capture:**

- raw logs, terminal output, or stack traces — summarize the conclusion instead
- code dumps and repo working files — the repo holds them, the vault links to them. **Exception — approved specs and plans:** when a design or plan is approved, store a snapshot in the vault at `<project folder>/Specs/` or `<project folder>/Plans/` (create on first use), headed with the repo path, so the project's folder holds everything about the project
- secrets, credentials, API keys, or tokens — never write these into notes unless the user explicitly keeps such material in their vault and asks you to
- transient scratch work with no tomorrow-value
- long prose walls in daily notes

**Daily notes are an index, not a record:** one `- ` bullet per event, about 20 words, ending with a `[[wikilink]]` to the note that holds the detail. `jotura today --append` writes your text verbatim, so the `- ` is yours to include.

## Linking

Every note connects. End notes with a `## Links` section of `[[wikilinks]]` to the project, people, and topic notes they relate to; an unlinked note is a dead end. Wikilinks resolve by filename (case-insensitive, shortest path) and then by frontmatter `aliases`: moves are safe, renames break links unless the old name is kept as an alias (check `jotura backlinks` before renaming).

## Agent config lives in `Agents/`

The vault's `Agents/` folder is the canonical home for agent configuration — global instruction files, skills, subagent definitions. Edit the vault copy, then run `jotura agents sync` to write it out to the system locations; never hand-edit both sides in one session. A file changed on both sides is skipped as a conflict (exit 16) — resolve it with `jotura agents sync --prefer vault|system`.

## Where learned knowledge lives

Not here. This file is a shipped artifact that upgrades with the CLI — anything written into it can be replaced. The durable, cross-agent home for everything learned about this vault is the Agent Conventions note (`jotura conventions`): folder layout, project structure, naming conventions, what is work vs personal, and the log of the user's filing corrections.
