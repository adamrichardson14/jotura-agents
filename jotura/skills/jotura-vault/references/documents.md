# Documents and plaintext files

## Plaintext files

Vaults hold more than markdown. Plaintext files (`txt`, `sql`, `log`, `json`,
`yaml`, `toml`, `xml`, `html`, `css`, `csv`, `tsv`, code files such as
`py`/`rs`/`ts`, shell scripts, `ps1`, `bat`, `ini`, `conf`, `env`, and
Obsidian's JSON `canvas` and `base` files) are
first-class: they appear in `ls`, are indexed by `search`, scanned by `grep`,
sync like notes, and are read and edited with the same commands and the same
hash-CAS workflow.

Two differences from `.md` notes:

- **No frontmatter semantics.** `read --json` returns the whole file as
  `body` with `frontmatter: null`, even when the file starts with `---`.
  Edit ops operate on the whole content. `frontmatter set/delete` and
  `tag add/remove` exit 9 (`InvalidArgs`) on non-markdown paths.
- **HTML files are plaintext, not documents.** They are indexed by their
  visible text (tags, styles, and scripts are stripped at index time) and
  get no markdown mirror. The desktop app shows them as a rendered,
  script-free preview with a source toggle.

## Working with documents

Besides notes and plaintext files, the vault holds **documents** (PDFs, Word
docs, spreadsheets, images). HTML files are NOT documents; they are plaintext
(no mirror is generated, and mirrors created by older versions are cleaned up
automatically by the desktop app). Documents
are stored as plain files just like notes, but their bytes aren't searchable
on their own. So each document gets a **searchable markdown mirror** in the
md_store at `.md_store/<original-path>.<ext>.md`, produced by a bundled
converter. The mirror is a normal note carrying the converted text; the
original document stays untouched.

Automatic conversion is a per-vault desktop setting ("Convert documents
automatically", stored in `<vault>/.jotura/vault-settings.json`). It is on
for ordinary vaults and off on first open for a vault that contains
`.obsidian/`, so an Obsidian vault's PDFs have no mirrors until the user
opts in. The CLI is unaffected: `import` converts explicitly (skip it with
`--no-convert`) and `regenerate-md-store` converts whatever you name. Do
not run `regenerate-md-store --all` in an Obsidian vault unless the user
asked for mirrors.

The md_store is excluded from `ls`, `quick-open`, and plain `search`, so it
never clutters note-level results.

```sh
# Import a document. Defaults to attachments/, then generates the md_store mirror.
jotura import ~/Downloads/contract.pdf --json
# → { "path": "attachments/contract.pdf", "contentHash": "...", "size": 12345,
#     "mdStorePath": ".md_store/attachments/contract.pdf.md", "converted": true }

# Choose a folder, or an explicit full vault path:
jotura import ~/Downloads/contract.pdf --folder projects/acme --json
jotura import ~/Downloads/contract.pdf --as projects/acme/signed-contract.pdf --json

# Import bytes from stdin (--as is required):
cat report.pdf | jotura import - --as attachments/report.pdf --json

# Skip mirror generation (store only the document itself):
jotura import data.bin --no-convert --json

# List documents with their conversion status:
jotura ls-documents --json
jotura ls-documents --folder attachments --json   # scoped to one folder
# → [{ "path": "attachments/contract.pdf", "size": 12345,
#      "hasMdStore": true, "conversionOk": true }]

# Regenerate mirrors (after editing a document, or to retry a failed conversion):
jotura regenerate-md-store attachments/contract.pdf --json
jotura regenerate-md-store --all --json
jotura regenerate-md-store --stale-only --json   # only the ones that need it
```

If the converter isn't installed yet, `import` still succeeds: it writes a stub
mirror with `converted: false` and a `note` field saying conversion was
deferred. The mirror self-heals once a converter is present (`regenerate-md-store
--stale-only`).

Need the raw bytes of a document? It's a plain file: read it straight from
the vault directory (`<vault>/attachments/contract.pdf`).

### Clipping web pages

`jotura clip` saves a web page as a markdown note. It is not an import: no file
is stored and no converter runs. The CLI fetches the URL itself, keeps the
article body while dropping navigation, headers, and footers, converts that to
markdown, and writes a normal note under `Clippings/` with `title`, `source`,
and `clipped` frontmatter. The source may also be a saved `.html` file on disk,
which is the fallback for pages that need a signed-in browser to render.

Only `http` and `https` URLs are fetched (anything else exits 9), the fetch
times out after 15 seconds, and a body over 8 MiB is refused (exit 1) with
advice to save the page and clip the file. The filename is a slug of the
extracted title; `--folder` suffixes on collision the way `create` does
(`page.md`, `page (2).md`), while `--as` names an exact path and refuses to
overwrite without `--force` (exit 9). Extraction is heuristic and
runs no JavaScript, so a page whose text is assembled client-side may extract
poorly or fail with `no readable article`.

```sh
# Clip a page into Clippings/
jotura clip https://example.com/posts/plain-text --json
# → { "path": "Clippings/plain-text-beats-a-database.md", "hash": "...",
#     "title": "Plain text beats a database",
#     "source": "https://example.com/posts/plain-text" }

# Choose a folder, or an explicit full vault path:
jotura clip https://example.com/x --folder Reading/2026 --json
jotura clip https://example.com/x --as Reading/spec.md --force --json

# Clip a page you saved from the browser:
jotura clip ~/Downloads/article.html --json
```
