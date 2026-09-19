---
name: jotura-vault
description: Complete command and flag reference for the `jotura` CLI, which reads, searches, and edits the user's Jotura markdown vault. Invoke this BEFORE running any `jotura` command — including the first one, and including commands you believe you already know. The flag grammar is not guessable: body-supplying ops such as `--append` and `--insert-after` take `--content`, while `--replace` takes `--with`, and the wrong pairing is rejected rather than applied. Also covers hash-CAS conditional writes, unique-match replaces, semantic search modes, documents, trash, templates, daily notes, agent config sync, and every exit code. Triggers on "Jotura", "my vault", "my notes", "my journal", "daily note"; on any question whose answer lives in personal notes; and on any request to create, read, search, rename, delete, tag, or modify a note. Do NOT use this skill for files outside the user's vault.
---

# Jotura vault

The user keeps their notes in a Jotura markdown vault, a plain directory of `.md` notes, plaintext files (code, config, SQL, HTML, logs), and imported documents. You access it through the `jotura` CLI binary (`~/.cargo/bin/jotura`), which uses the same Rust core as the Jotura desktop app and guarantees atomic writes, round-trip Markdown fidelity, and hash-based conflict detection. Cloud sync (when the user has enabled it) is end-to-end encrypted, but on disk everything is plaintext: there is nothing to unlock.

The user often runs the Jotura desktop app concurrently. The CLI and desktop coordinate automatically: the desktop's file watcher picks up any change you make within ~1 second, and the CLI's `--if-hash` checks protect you from clobbering the desktop's writes.

You MAY read vault files directly (`cat`, `grep`, editors), but prefer the CLI for every mutation: it enforces unique-match replaces, preserves frontmatter, writes atomically, and honors `--if-hash` concurrency checks.

Two skills ship together: this one (CLI mechanics) and **jotura-notes** (filing and capture — where notes go, what belongs in them, when to write proactively). Consult jotura-notes before deciding where any content goes. Everything agents learn about a specific vault lives in that vault's Agent Conventions note (`jotura conventions`), not in either skill. `jotura skill install` writes both SKILL.md files as a managed block, so your own content around the markers survives every upgrade; the files under `references/` are plain files that get replaced wholesale on the next version bump or `--force`, so never edit them by hand.

## Before you do anything

Run `jotura status --json` first (or `jotura doctor --json` for a deeper health check). It returns one of:

```json
{"vault": "/path/to/vault", "noteCount": 247, "sync": {"enabled": true, "keyCached": true, "paused": false}}
{"error": {"code": "NoVault", ...}}
```

- A `vault` path → you can proceed. Data commands never need a password.
- `NoVault` → ask the user to set `JOTURA_VAULT=/path/to/vault` in their shell, pass `--vault` to commands, or open a vault in the desktop app first.
- The `sync` block is informational: `enabled: false` means the vault is local-only; `keyCached: false` on a sync-enabled vault means uploads are paused until the user provides their sync password (`jotura sync login`, or unlock in the desktop). Reads and writes always work regardless.

## The canonical safe-edit workflow

Every edit should follow this pattern. The `--if-hash` check stops you from clobbering work the user did in the desktop app, in another CLI invocation, or that arrived via sync.

```sh
# 1. Read with a hash
HASH=$(jotura read "<path>" --json | jq -r .hash)

# 2. Edit using the hash as a precondition
jotura edit "<path>" \
  --replace "<exact old text>" \
  --with "<new text>" \
  --if-hash "$HASH"
```

If exit code is 2 (`HashConflict`), the file changed under you. The error JSON carries `path`, `expectedHash`, `actualHash` and a `currentVsIntended` unified-diff between what's on disk *right now* and what you were about to write, so you can decide whether to retry, merge, or abandon without re-reading.

### Previewing edits with `--dry-run`

Every mutating command (`write`, `edit`, `create`, `rename`, `delete`, `frontmatter set/delete`, `tag add/remove`, `batch`) accepts `--dry-run`. The command computes exactly what it would write, prints a structured preview, and does NOT touch disk or manifest. Use this when you're not sure an edit will hit the right text:

```sh
jotura edit notes/today.md --replace "TODO" --with "DONE" --dry-run --json
# → { "dryRun": true, "currentHash": "...", "newHash": "...", "changes": 1,
#     "preview": { "kind": "replace", "before": "TODO", "after": "DONE" } }
```

Exit codes still fire as expected: a dry-run that *would have* failed with `HashConflict` (code 2) or `Ambiguous` (code 4) returns the same error without writing anything.

### Seeing the diff with `--diff`

Every mutating command also accepts `--diff`. The JSON output gets an extra `"diff"` field containing a standard unified-diff between the current and new content (3 lines of context, `---`/`+++`/`@@` headers). Combine with `--dry-run` to preview without writing:

```sh
jotura edit notes/today.md --replace "TODO" --with "DONE" --diff --dry-run --json
# → { ..., "diff": "@@ -3,3 +3,3 @@\n line a\n-TODO\n+DONE\n line c\n" }
```

For `create`, `rename`, and `delete` the diff is structural (a `(file renamed)` / `(file created)` / `(file deleted)` marker block) rather than a real text diff. There's no old-vs-new body to compare.

## Reading

```sh
jotura read inbox/today.md                     # body only, the common case
jotura read inbox/today.md --numbered          # line numbers, before a line-based edit
jotura read inbox/today.md --json              # { path, hash, frontmatter, body, lineCount }
jotura read inbox/today.md --with-frontmatter  # include the YAML block
```

A bare name with no slash or extension resolves like a wikilink to the unique note with that filename; several matches exit 4 with `candidates`, so prefer the path when you know it. `read` on an image or other binary file describes it instead of printing bytes: `--json` returns `{ path, absolutePath, kind: "image" | "binary", mime, bytes }`, and you open `absolutePath` with your own image-capable file tool to see the picture. Read `references/images.md` before working with images.

## Finding notes

```sh
jotura search "deadline next week" --json          # full-text; --in projects/ scopes it
jotura search "why did we pick sqlite" --mode smart --json   # keyword + meaning, fused
jotura quick-open "meeting" --json                 # filename + frontmatter-alias fuzzy match, much faster
jotura grep "TODO" --json                          # regex over bodies
jotura backlinks projects/acme.md --json           # notes linking TO this one
```

`jotura search` builds an in-memory index over the vault on every call (it uses Tantivy internally, but nothing is persisted between invocations). `quick-open` matches filenames and frontmatter `aliases`; a hit found through an alias carries an `alias` field in `--json` output. `jotura ls --json` lists every note, `--folder P` one folder. Read `references/search.md` before using semantic or smart mode, the `grep` flags, `links`, or document search.

## Editing: string-based (preferred)

This is the safest primitive. Strict by default: fails if the old text isn't unique. Mirrors Claude Code's own `Edit` tool semantics.

```sh
# Single targeted change - fails with code 4 if "TODO: reply to alice" appears more than once
jotura edit notes/today.md \
  --replace "TODO: reply to alice" \
  --with "DONE: replied to alice" \
  --if-hash "$HASH"

# Replace every occurrence (use when you mean it)
jotura edit notes/today.md --replace "TODO" --with "DONE" --all

# Replace the Nth occurrence (1-indexed) - use when --replace returns code 4 (Ambiguous)
jotura edit notes/today.md --replace "TODO" --with "DONE" --nth 2
```

When `--replace` fails with exit code 4 (`Ambiguous`), stderr JSON includes `lineHits: [12, 47, 83]`. Use those numbers to pick the right `--nth`, or extend the old-text to include enough surrounding context that it becomes unique.

## Editing: line-based (when string matching doesn't fit)

Get the line numbers from `jotura read --numbered` first. Line numbers are 1-indexed and refer to the **body** of the note (frontmatter is stripped from the count).

```sh
jotura edit notes/today.md --replace-line 12 --with "new content for line 12"
jotura edit notes/today.md --replace-lines 12:15 --with $'new\nmulti-line\ncontent'   # real newlines: the CLI takes text verbatim
jotura edit notes/today.md --insert-after 12 --content "new line inserted"
jotura edit notes/today.md --insert-before 1 --content "new first line of body"
jotura edit notes/today.md --delete-lines 12:15
jotura edit notes/today.md --append --content "appended at end"        # append-by-design notes only, see below
jotura edit notes/today.md --prepend --content "prepended after frontmatter"
```

`--content` is also readable from stdin: `echo "stuff" | jotura edit ... --insert-after 5`.

Text flags are taken verbatim: `"a\nb"` writes a literal backslash-n. For multi-line text use `$'a\nb'` quoting, pipe it on stdin (insert/append/prepend), or use `--apply` with a JSON array where strings carry real newlines.

`--append` and `--prepend` are the exceptions, not the defaults. Read "Adding content to an existing note" below before using either.

**The two text flags are not interchangeable.** `--with` carries the new text for `--replace`, `--replace-line` and `--replace-lines`; `--content` carries it for `--insert-before`, `--insert-after`, `--append` and `--prepend`; `--delete-lines` and `--apply` take neither. Passing the wrong one exits 9 (`InvalidArgs`) before the note is read, so nothing is written. Content that resolves to empty (an `--content ""`, or an empty stdin) also exits 9 rather than writing a body that did not change.

## Adding content to an existing note

`--append` is not how content is added to a note. The default is: read the note, find the line the content belongs after, insert there, re-read to confirm. `--append` writes after the last line of the body, which in a well-formed note is below `## Links`, below everything.

```sh
# 1. Read the whole note with line numbers, and the hash for --if-hash
HASH=$(jotura read projects/acme.md --json | jq -r .hash)
jotura read projects/acme.md --numbered
#   14  ## Decisions ... 16  - 2026-09-02: per-file DEKs wrapped by the master key
#   18  ## Open questions ... 31  ## Links

# 2. Insert after the last line of the section the content belongs to
jotura edit projects/acme.md --insert-after 16 \
  --content "- 2026-09-06: SQLite for the sync db, one binary to ship" \
  --if-hash "$HASH" --json

# Multi-line content: omit --content and pipe it on stdin
jotura edit projects/acme.md --insert-after 16 --if-hash "$HASH" --json <<'EOF'
- 2026-09-06: SQLite for the sync db, one binary to ship
  - rejected Postgres: an extra service to run for a local-first app
EOF

# 3. Re-read the region to confirm placement
jotura read projects/acme.md --numbered | sed -n '14,20p'
```

If the note has no section for the content, insert a new heading in the note's style at the position it belongs (`--insert-after` the last line of the section that precedes it), still above `## Links`. When line numbers might have shifted between your read and your write, anchor on text instead: `--replace` the section's unique last line and pass `--with` that same line followed by the new lines (`$'...'` quoting for the real newlines). Text anchors are stable where line numbers are not.

Reserve `--append` for append-by-design notes: the daily note (`jotura today --append`) and logs whose convention is newest-at-bottom. Adding a `## Update` heading at the end of a note is an append with extra steps.

## Frontmatter

The `edit` command **never touches frontmatter**. Use these dedicated operations instead. Frontmatter changes are isolated so YAML can't accidentally be mangled.

```sh
jotura frontmatter get notes/today.md          # full YAML block
jotura frontmatter get notes/today.md status   # one key
jotura frontmatter set notes/today.md status "in-progress"
jotura frontmatter delete notes/today.md draft
jotura tag add notes/today.md project-x        # tag helpers work on the `tags:` array
jotura tag remove notes/today.md old-tag
jotura tag list notes/today.md
```

The frontmatter editor handles top-level scalars and `tags:` arrays. Complex nested YAML (anchors, multi-line scalars, nested maps) is preserved verbatim but can't be edited from the CLI. For those, ask the user to edit in the desktop.

## Daily notes

```sh
# Today's daily-note path, auto-creating if missing
jotura today --json
# → { "path": "daily/2026/05/26.md", "hash": "...", "created": true }

jotura today --read                       # today's body
jotura today --date 2026-05-20 --read     # a different date

# Append a line (also reads from stdin if --append has no value).
# The text is written verbatim: include the "- " yourself or the entry is not a bullet
jotura today --append "- took a 20-minute walk after lunch [[Walking]]"

# Override the folder. Without --folder, an existing case-variant of the daily
# folder (for example `Daily/`) is reused so the tree never splits into two
# spellings across case-sensitive devices; `daily` is used only when none exists.
jotura today --folder Journal
```

Convention: `<folder>/YYYY/MM/YYYY-MM-DD.md`. When the vault has `.obsidian/daily-notes.json`, `today` follows that file instead (its `folder`, `format`, and `template` keys, same as the desktop app), so existing Obsidian daily notes and new ones share a layout; the supported format tokens are `YYYY YY MM M DD D` and a format using anything else falls back to the convention above inside that folder. The CLI's `today` is path computation + auto-create of the one requested day; `--append` adds no bullet marker, indentation, or blank line, so pass the finished line. Pre-creating a window of upcoming days is a desktop setting ("Create upcoming daily notes", off by default), so do not expect tomorrow's note to exist. Once you know the path, `edit` and `read` work the same as for any other note.

`jotura tasks --json` lists every task the desktop Tasks pane would show; `--all` drops the 30-day window.

## Exit codes: handle these by reading the JSON error on stderr

| Code | Name | Recovery |
|---|---|---|
| 0 | success | proceed |
| 2 | `HashConflict` | error includes `currentVsIntended` unified diff; inspect it to decide retry / merge / abandon. For `batch`, comes as a `conflicts: [...]` array of per-path entries |
| 3 | `NotFound` | the string you tried to replace isn't there, or the path is wrong. Re-read and check |
| 4 | `Ambiguous` | parse `lineHits` from the JSON error; pick `--nth N` or extend `--replace` text for uniqueness |
| 5 | `OutOfRange` | line number exceeds `lineCount`; re-read with `--numbered` |
| 6 | `NoVault` | ask the user to set `JOTURA_VAULT` or open a vault in the desktop |
| 8 | `PermissionDenied` | surface the message to the user |
| 9 | `InvalidArgs` | bad flag combination; re-check `--help` |
| 11 | `MultiEditOpFailed` | one op in a `--apply` or `batch` failed; check `opIndex` + `underlyingCode` to identify which op and why; the whole batch was aborted with no writes |
| 12 | `SyncNotEnabled` | `sync login`/`sync logout` on a vault without sync configured; only relevant to sync key management |
| 13 | `SemanticUnavailable` | `search --mode semantic\|smart` isn't ready: the message says whether the index, model, runtime, or (OpenAI indexes) API key is missing; fall back to `--mode keyword` and ask the user to enable semantic search in the desktop app (Settings → Semantic search → Enable) or set `OPENAI_API_KEY` |
| 15 | `ReadOnlyShare` | the target is inside a viewer-role shared mount (`path`, `shareId`, `mountRel` in the error JSON); writes are refused with no override, ask the share owner for editor access. Mount-structure operations (deleting or renaming a mount root or a folder containing one) come back as `InvalidArgs` (code 9) directing you to the desktop app's share management |

Errors are emitted as a single-line JSON object on stderr, e.g. `{"code":"Ambiguous","message":"...","needle":"TODO","occurrences":3,"lineHits":[5,12,47]}`.

## Common mistakes to avoid

1. **Prefer `jotura` over `sed`/`awk`/direct writes for mutations.** Vault files are plain `.md` and direct edits won't corrupt anything, but they skip the safety rails: no `--if-hash` protection against concurrent desktop/sync edits, no unique-match strictness, no frontmatter preservation guarantees, no atomic write.
2. **Never write into `<vault>/.jotura/`** (except templates via the `templates` commands). It holds the sync database, search index, and trash. The CLI and desktop manage it.
3. **Always use `--if-hash` on writes** of notes the user might be actively editing. Without it, you can silently clobber their desktop work.
4. **Always use `--json` for structured output** when you'll act on the result programmatically. Default output is human-readable.
5. **Don't use `--replace --all` defensively.** Strict mode catches "I thought there was one but there were five" mistakes. Use `--all` only when the user explicitly asked for "every occurrence."
6. **Don't whole-body-write when you mean to edit a section.** `jotura write` replaces the body wholesale. Use `jotura edit --replace` to surgically change one place.
7. **Don't `--append` to add content to a note.** Appending puts the content after `## Links`, at the bottom of a document the user organized by section. Read the note, then `--insert-after` the last line of the section it belongs to (see "Adding content to an existing note"). `--append` is for the daily note and other append-by-design logs only.
8. **Don't run `jotura sync login` yourself.** It needs the user's sync password. If sync uploads are paused for a missing key, tell the user to run it (or unlock in the desktop). Your reads and edits work fine regardless.

## Quick reference (memorize this)

```
status / doctor   → vault + sync state / full health
sync status|login|logout → sync key cache management
ls --json         → list notes
search Q --json   → full-text search
search Q --in P   → full-text search scoped to a path prefix
search Q --mode semantic|smart → meaning-based / fused (exit 13 if not set up)
quick-open Q      → filename and alias search
import FILE [--folder F | --as P] [--no-convert]  → add a document + md_store mirror
clip URL [--folder F | --as P]                → save a web page as a note (see references/documents.md)
search-documents Q / ls-documents [--folder F] / regenerate-md-store → documents (md_store)
read P --numbered → read with line numbers
read P --json     → read with hash for --if-hash; an image or binary gives { absolutePath, kind, mime, bytes } instead (references/images.md)
edit P --replace OLD --with NEW --if-hash H   → safe targeted edit
edit P --apply FILE --if-hash H               → multi-op edit, sequential, atomic
edit ... --dry-run / --diff                   → preview / unified diff
edit P --replace-line N --with C              → line-based edit
edit P --insert-after N --content C           → add to an existing note: inside its section (the default)
edit P --append --content C                   → --content, never --with; append-by-design notes only
batch FILE [--dry-run] [--diff]               → atomic multi-file ops
frontmatter get/set/delete P [K] [V]          → YAML ops (.md only, else exit 9)
tag add/remove/list P [T]                     → tags (.md only, else exit 9)
tags [--notes T]                              → every tag with counts / notes carrying T; counts frontmatter and inline body tags, `tag list P` reads the frontmatter block only
create PARENT NAME [--template T]              → new note (optionally from template)
rename A B / delete P                          → rename / permanent delete
trash P / trash list / restore P [--rename-to N]  → soft-delete, list, restore
trash purge P / trash empty --force            → permanent removal
templates list/show/create/delete              → manage templates
today [--read] [--append T] [--date D] [--template T] → daily-note helper
tasks [--all] [--section S] [--open|--done]   → grouped task list (matches the Tasks pane)
conventions       → the Agent Conventions note: the agents' shared memory of
                    this vault's filing rules (auto-creates)
context [TASK]    → context pack at task start: conventions + ranked relevant
                    memories + open follow-ups (session digest without TASK)
memory log --kind decision|fact|followup --title T [--occurred D]
                  → log memory; supersede similar: hits. --occurred = when true
memory recall Q [--mode keyword|semantic|smart] [--kind K --project P
                --include-stale]  → ranked memory search, smart by default
memory list [--kind K --status S --stale]  → browse memory
memory supersede OLD --by NEW / memory resolve P → close the loop
memory doctor [--fix]  → stale actives, stalled follow-ups, broken edges
hook install claude → SessionStart hook injecting `jotura context`
backlinks P [--limit N]                        → notes linking TO P
links P [--limit N]                            → links inside P (outgoing)
grep PAT [-i -x -w -c -l -L] [--include G --exclude G] [--max-count N]
                                               → regex search across bodies
watch [--paths G] [--kinds CMDR] [--once]      → stream events
completion <shell>                             → emit shell completion
skill install <agent>                          → install this skill
Jotura/theme.json → custom desktop theme (references/lifecycle.md); normal edit loop
```

## References (read the one you need)

| File | Read it before |
|---|---|
| `references/search.md` | semantic or smart search, grep flags, backlinks and links, searching documents |
| `references/documents.md` | importing PDFs, Word, spreadsheets; plaintext and HTML files; md_store mirrors |
| `references/batch.md` | more than one edit in a note (`--apply`) or across notes (`batch`) |
| `references/lifecycle.md` | create, rename, trash and restore, templates, whole-body writes, watch, completion, doctor |
| `references/agents.md` | the vault's `Agents/` folder, `agents sync`, `skill install`, the SessionStart hook |
| `references/images.md` | looking at an image a note embeds, adding an image to a note, the `![[img.png\|300]]` sizing syntax |
