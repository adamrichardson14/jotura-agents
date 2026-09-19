# Finding notes in depth

## Semantic search: `--mode semantic|smart`

`jotura search` takes a `--mode` flag with three values. **Semantic-backed
search is the preferred mechanism: default to `--mode smart` for every note
search.** The flag's own default, `keyword`, is the ranked full-text search
described for `jotura search` in SKILL.md; treat it as the fallback for when
semantic is unavailable (exit 13), not as the first choice. The two
semantic-backed modes:

```sh
# semantic: embedding similarity - phrasing can differ from the note
jotura search "how do I renew the TLS certificate" --mode semantic --json

# smart: keyword + semantic merged by rank fusion - best default for
# open-ended or exploratory questions
jotura search "what did I decide about hosting" --mode smart --json
```

- `smart`: runs keyword and semantic and merges them with reciprocal-rank
  fusion, so a note ranked by both lists rises to the top. **The preferred
  default** - it never loses an exact match and adds meaning-based hits.
- `semantic`: meaning and paraphrase ("notes about X" where X is a concept,
  not a phrase). One result per note; the best-matching section wins.
- `keyword`: exact/literal terms. Fastest; always reflects the files on disk.
  The fallback when semantic is unavailable, and the right tool only when a
  result must reflect this second's files. For exact regex matching use
  `grep` instead.

Semantic/smart JSON output: `[{ "path", "headingPath", "snippet", "score" }]`
with vault-relative paths (text output is `path<TAB>score<TAB>snippet`).

**Prerequisites.** Semantic search must have been enabled in the desktop app
at least once (Settings → Semantic search → Enable). That downloads the
embedding model and builds the vector index. The CLI only *queries*; it never
indexes.

**Freshness caveat.** The index is maintained by the desktop app. Notes you
write via the CLI become semantically searchable after the app next indexes
them (about a second later while the app is running; on next launch
otherwise). Keyword mode always reflects the current files.

**Embedding provider.** The index's provider (local model or OpenAI) is
chosen in the desktop app (Settings -> Semantic search) and the CLI embeds
queries with whichever one built the index. An OpenAI index needs an API key:
`OPENAI_API_KEY` in the environment, or the key saved in the desktop app.
`jotura doctor` shows the provider under `semantic-index` and, for OpenAI
indexes, a `semantic-key` check (key source only, never the key).

**Exit code 13 (`SemanticUnavailable`).** The JSON error says exactly what's
missing: no index, index disabled, embedding model absent, ONNX runtime
unavailable, or (OpenAI indexes) no usable API key. The two key messages are
`semantic index uses OpenAI embeddings; set OPENAI_API_KEY or add the key in
the desktop app (Settings -> Semantic search)` and `OpenAI rejected the API
key; update it in the desktop app or OPENAI_API_KEY`. Recovery: fall back to
`--mode keyword` (always works) and tell the user to enable semantic search
in the desktop app (Settings → Semantic search → Enable) or fix the key.
Don't retry semantic modes until they have. A network or rate-limit failure
from OpenAI is exit 1 with a redacted message; retrying later is reasonable.

## Exact-match search with `grep`

`jotura grep <pattern>` runs a regex over every text file body (markdown
notes and plaintext files alike). It's the
right tool when you need exact, predictable matches. `search` is fuzzy /
ranked and `quick-open` only looks at filenames.

```sh
# basic regex match - output is path:line:matched-text per match
jotura grep 'TODO[:\s]'

# case-insensitive
jotura grep --ignore-case 'unicorn'

# whole-line match
jotura grep --line-regexp '^# Daily'

# whole-word match
jotura grep --word-regexp 'jot'

# restrict to a folder via include glob
jotura grep 'TODO' --include 'inbox/*'

# only paths that contain a match
jotura grep 'TODO' --files-with-matches

# count matches per file
jotura grep 'TODO' --count

# stop after N matches per file
jotura grep 'TODO' --max-count 3

# JSON output: [{ path, line, lineText, matchedText }]
jotura grep 'TODO' --json
```

Flags: `-i`/`--ignore-case`, `-x`/`--line-regexp`, `-w`/`--word-regexp`,
`-c`/`--count`, `--max-count N`, `--include GLOB`, `--exclude GLOB`,
`-l`/`--files-with-matches`, `-L`/`--files-without-match`.

Templates (`.jotura/templates/`) and trashed notes are excluded
automatically. Cost: every note matching `--include` is read and
scanned (sub-second for a few thousand notes, several seconds for 10k+).
For fuzzy/ranked text search use `search`; for filename lookup use
`quick-open`.

## Finding links between notes

```sh
# List notes that link TO this path (backlinks)
jotura backlinks inbox/today.md --json

# List links inside a note (outgoing)
jotura links projects/x.md --json

# Cap results
jotura backlinks inbox/today.md --limit 20
```

Two link syntaxes are recognized:

- **Wikilinks:** `[[target]]`, `[[target|display text]]`, and embeds `![[target]]`
- **Markdown links:** `[display](target.md)` (the target must end in `.md`;
  percent-encoding such as `My%20Note.md` is decoded)

`#heading` and `#^block` suffixes (`[[foo#heading]]`, `[[foo#^id]]`,
`[foo](foo.md#h)`) are stripped before matching. `[[#heading]]` is the note
the link is written in.

Targets resolve the way Obsidian resolves them, through the resolver the
desktop app shares (`jotura_core::links`), so both surfaces agree:

1. **Case-insensitive throughout.** `[[roadmap]]` matches `Roadmap.md`.
2. **Path match** when the target has a slash. `[[inbox/today.md]]` and
   `[[inbox/today]]` both match the note at `inbox/today.md`.
3. **Shortest-path filename match** for a bare name. `[[today]]` matches
   the `today.md` nearest the vault root; equal-depth ties all match, so
   *both* show up as backlinks for either one.
4. **Alias match.** A name that matches no filename is checked against
   frontmatter `aliases` (a list, an inline array, or a single scalar), so
   `[[The Plan]]` matches a note carrying `aliases: [The Plan]`. The first
   note by path wins a shared alias.
5. **No fuzzy fallback.** A target matching none of the above is
   `(unresolved)`; nothing guesses a nearby name.

Embeds of non-markdown files (`![[diagram.png]]`) resolve by filename, then
through the Obsidian attachment folder (`attachmentFolderPath` in
`.obsidian/app.json`), then the note's own folder. Title-based resolution
(matching frontmatter `title:`) is still **not** implemented.

The desktop editor treats the same syntax as first-class and keeps it
byte-for-byte: `[[Note|alias]]`, `[[Note#Heading]]`, `[[Note#^block]]`,
`![[image.png|300]]`, callouts `> [!kind] Title` / `> [!kind]-`, task
states `[/] [-] [>] [?]` beside `[ ]`/`[x]`, `==highlight==`, `$math$`,
`%% comment %%`, footnotes `[^1]`, block ids `^id`. Write them as plain
text; no escaping is needed. Only `[ ]`, `[x]`, `[X]` count as tasks in the
desktop task list.

Implementation: both commands scan every note per invocation
(brute force, no persistent index). Sub-second for typical vaults; expect
seconds for 10k+ notes.

## Searching documents

When the user asks to search their documents (PDFs, Word docs, spreadsheets,
etc.), as opposed to their notes, use `jotura search-documents <query>`. It
searches the markdown representations in the md_store and returns the ORIGINAL
document paths. Do NOT use plain `jotura search` for documents; that searches
notes and excludes the md_store. Do NOT try to read PDF/docx bytes directly;
read the converted markdown via `jotura read .md_store/<path>.<ext>.md` or rely
on search-documents snippets.

```sh
jotura search-documents "quarterly revenue" --json
# → [{ "documentPath": "attachments/q3-report.pdf",
#      "mdStorePath": ".md_store/attachments/q3-report.pdf.md",
#      "title": "q3-report", "snippet": "...quarterly <mark>revenue</mark>...",
#      "score": 0.92 }]
jotura ls-documents --json
jotura read ".md_store/attachments/q3-report.pdf.md"   # read the converted text
```

`search-documents` returns the original `documentPath` (e.g.
`attachments/q3-report.pdf`), not the `.md_store/...` mirror path, so you can
hand it straight back to the user or read the file directly.
