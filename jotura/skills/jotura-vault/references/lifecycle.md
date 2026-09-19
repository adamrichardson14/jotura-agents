# Note lifecycle

## File operations

```sh
jotura create inbox "meeting-notes"   # creates inbox/meeting-notes.md (auto-suffixes (2) on collision)
jotura rename old/path.md new/path.md
jotura trash inbox/old.md             # SOFT-DELETE (recoverable, recommended)
jotura delete inbox/old.md            # PERMANENT delete - escape hatch only

# `jotura delete --trash <path>` is an alias for `jotura trash <path>`.
```

For agent use, prefer `jotura trash`: it's reversible. `jotura delete`
permanently removes the file with no recovery path.

## Soft-delete with `trash`

Soft-deletion moves the file into `<vault>/.jotura/trash/`, keeping its
relative path and a small metadata sidecar. The note becomes invisible to
`ls`, `read`, `search`, `quick-open`, and `grep`, but it can be restored at
any time. The desktop app uses the same trash directory.

```sh
# Soft-delete a note
jotura trash inbox/old.md

# List everything in the trash (sorted newest-first)
jotura trash list --json
# → [{ "originalPath": "...", "isFolder": false, "size": 123, "trashedAt": 1716... }]

# Restore at the original path (fails if a note now exists there)
jotura trash restore inbox/old.md

# Restore at a different path
jotura trash restore inbox/old.md --rename-to archive/old.md

# Permanently delete one trashed item
jotura trash purge inbox/old.md

# Permanently delete every trashed item (requires --force)
jotura trash empty --force
```

The original path is the key for `restore` and `purge`: pass the same
logical path the note had before it was trashed. `jotura trash list` shows
those keys.

Restoring into a path that's already occupied returns `InvalidArgs` (exit
code 9) with a message suggesting `--rename-to`. Folders can be trashed and
restored whole.

## Templates

Reusable note bodies live at `.jotura/templates/<name>.md` inside the
vault (plain files, but excluded from `ls`/`search`/`quick-open`/`grep`).
A vault with `.obsidian/templates.json` also has an Obsidian templates
folder: `templates list` and `templates show` include it, but it is
read-only here. `templates create`/`delete` only touch `.jotura/templates/`;
edit an Obsidian template as an ordinary note.

```sh
# List available templates
jotura templates list

# Print a template's body
jotura templates show meeting

# Create a template from stdin
echo "# {{title}}\n\nDate: {{date}}\n" | jotura templates create meeting

# Create a template from an existing note's body
jotura templates create from-current --from inbox/current.md

# Delete a template
jotura templates delete meeting
```

### Using a template

Pass `--template <name>` to `create` or `today` to initialize the new
note's body from a template. Existing notes are never overwritten: the
flag only affects newly-created notes.

```sh
jotura create inbox meeting-notes --template meeting
jotura today --template daily
```

### Template variables

Simple find-replace (no conditionals, loops, or partials). Both the
frontmatter block and the body have variables substituted.

| Variable | Substitution |
|---|---|
| `{{date}}` | today's date, `YYYY-MM-DD` (UTC) |
| `{{time}}` | current time, `HH:mm` (UTC) |
| `{{datetime}}` | current ISO 8601 timestamp |
| `{{date:FMT}}` / `{{time:FMT}}` | custom format from the tokens `YYYY YY MM M DD D HH H mm ss`, e.g. `{{date:DD/MM/YYYY}}` (Obsidian's syntax); any other character is copied through |
| `{{filename}}` | the new note's filename without `.md` |
| `{{title}}` | the filename with `-`/`_` turned to spaces, title-cased |

`{{date}}` always reflects the **current** date, even when `today --date
<past>` is used: that flag controls the path, not the variable.

## Whole-body writes (rare - prefer `edit`)

Only use this when the user genuinely wants to replace a note's entire body. Reading-then-editing with `edit --replace` is almost always better because it preserves any content the user added that you might not have anticipated.

```sh
echo "new body content" | jotura write notes/today.md --if-hash "$HASH"

# --full replaces frontmatter too. Almost never the right move.
echo "---\nfoo: bar\n---\nbody" | jotura write notes/today.md --full --if-hash "$HASH"
```

A `write` whose stdin is empty exits 9 instead of clearing the note - a pipe
that produced nothing never means "empty this file". Pass `--content ""` when
emptying it is what you actually want.

## Reacting to vault changes

`jotura watch` streams change events as newline-delimited JSON. Useful when scripting against the vault while the desktop or another agent is also editing.

```sh
jotura watch
# {"kind":"Created","path":"inbox/new.md","ts":"2026-05-26T17:50:00Z"}
# {"kind":"Modified","path":"daily/2026/05/26.md","ts":"..."}

# Filter to a glob of paths
jotura watch --paths "daily/**"

# Filter event kinds (C,M,D,R)
jotura watch --kinds C,M

# Exit after the first matching event
jotura watch --once
```

Events are detected by polling the file tree about once per second; renames surface as a Deleted + Created pair.

## Shell completion

The CLI ships completion scripts for bash, zsh, fish, elvish, and powershell:

```sh
# zsh - install once
jotura completion zsh > "${fpath[1]}/_jotura"

# bash
jotura completion bash > /usr/local/etc/bash_completion.d/jotura

# fish
jotura completion fish > ~/.config/fish/completions/jotura.fish
```

After install, `jotura <TAB>` completes subcommands and flags. For path arguments, completion delegates to `jotura __complete-paths <prefix>`, which walks the vault's file tree. When no vault is selected the completer returns nothing silently (no error).

## Troubleshooting with `doctor`

`jotura doctor` runs a sequence of health checks: CLI on PATH, vault detected and readable, sync state, semantic search readiness (index, provider, model or API key, runtime), document conversion health, desktop app version, etc. Use this first when something feels off:

```sh
jotura doctor --json
```

Each check returns `{ name, status: pass|warn|fail|info, detail }`. Doctor never mutates state.

## What the CLI deliberately does NOT do

Delegate these to the desktop app:

- Create a new vault (any folder works, but the desktop is the usual flow)
- Enable sync / set or rotate the sync password / restore from the server
  (the CLI only manages the local key cache: `sync status|login|logout`)
- Sync push / pull loops
- Recently-opened note list (UI feature)
- Move-to-OS-trash (the CLI's `trash` keeps soft-deleted notes inside the
  vault so they're available across machines)
- Opening files in external apps

## Custom themes (`Jotura/theme.json`)

The desktop app's colours can be customised with one JSON file in the vault,
`Jotura/theme.json`. The user turns it on in Settings → Appearance → Custom
theme, which creates the folder and seeds the file from the built-in theme
they have selected (`jotura`, `graphite`, `fjord`, `moss`, or `contrast`).
You cannot enable it from the CLI, but once it exists you can edit it like any
other vault file, and every save applies live in the app and syncs to the
user's other devices.

The file layers over the selected built-in: anything you leave out falls
through, so a minimal theme is a few lines.

```json
{
  "name": "Ocean",
  "light": { "accent": "oklch(48% 0.14 200)", "accent-contrast": "#fff" },
  "dark": { "accent": "oklch(76% 0.13 200)" }
}
```

- `light` and `dark` are optional objects; at least one is required. A missing
  variant leaves that mode on the built-in.
- Keys are token names. Core: `bg`, `bg-sidebar`, `bg-hover`, `bg-active`,
  `fg`, `fg-muted`, `border`, `accent`, `accent-contrast`, `danger`. Everything
  else is derived from these or has a per-mode default and may be overridden by
  name: `card-bg`, `hairline`, `control-bg`, `control-border`, `overlay`,
  `shadow`, `warning-bg`/`warning-border`/`warning-fg`, `editor-gutter-bg`,
  `editor-selection-bg`, the `editor-syntax-*` set (heading, strong, emphasis,
  link, string, keyword, comment, number, meta, code, quote, list,
  punctuation), the `diff-*` set, and the `badge-*` set.
- Values are any CSS colour: hex, `rgb()`, `hsl()`, `oklch()`, `color-mix()`.
  Prefer `oklch(L% C H)`: keep grounds and ink on one hue at low chroma and put
  the colour in `accent`. Keep `fg` at least 7:1 against `bg`, `fg-muted` and
  `accent` at least 4.5:1 against `bg`, and `accent-contrast` at least 4.5:1
  against `accent`, or the theme will be hard to read.
- Unknown keys and unusable values are skipped (the app lists them in
  Settings). Invalid JSON makes the app ignore the whole file, so validate
  before writing.

Edit it with the normal safe loop, treating it as a plaintext file (no
frontmatter):

```sh
HASH=$(jotura read Jotura/theme.json --json | jq -r .hash)
jotura edit Jotura/theme.json --replace '"accent": "oklch(50% 0.15 55)"' \
  --with '"accent": "oklch(48% 0.14 200)"' --if-hash "$HASH"
# or replace the whole palette
jotura write Jotura/theme.json --if-hash "$HASH" < theme.json
```

If `jotura read Jotura/theme.json` exits 3 (`NotFound`), the user has not
turned the switch on yet; ask them to enable Custom theme in Settings rather
than creating the file yourself, so the seed and the setting land together.
