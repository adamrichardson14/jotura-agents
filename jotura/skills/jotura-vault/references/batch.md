# Multi-edit and batch

## Editing: multi-edit in one call

`jotura edit ... --apply <FILE>` (or `-` for stdin) takes a JSON array of edit ops and applies them **sequentially to the evolving content**. One `--if-hash` precondition covers the whole batch; the final result is committed in a single atomic write.

```sh
cat > /tmp/edits.json <<'JSON'
[
  {"kind": "replace", "old": "TODO: x", "new": "DONE: x"},
  {"kind": "replace", "old": "TODO: y", "new": "DONE: y", "all": true},
  {"kind": "replace-line", "line": 12, "content": "new line 12"},
  {"kind": "insert-after", "line": 20, "content": "after 20"},
  {"kind": "delete-lines", "start": 25, "end": 27},
  {"kind": "append", "content": "end"}
]
JSON

jotura edit notes/today.md --apply /tmp/edits.json --if-hash "$HASH" --json
# or via stdin:
jotura edit notes/today.md --apply - --if-hash "$HASH" --json < /tmp/edits.json
```

Supported `kind` values: `replace`, `replace-line`, `replace-lines`, `insert-before`, `insert-after`, `delete-lines`, `append`, `prepend`. Field names match the matching CLI flag (`new` instead of `with`).

**Important: line numbers shift as ops apply.** Each op observes the state produced by every prior op. If op 0 deletes lines 1–3, then op 1's `line: 5` refers to what was originally line 8. Plan your edits with this in mind, or use string-based replaces which are stable across shifts.

**Atomic on failure.** If any op fails (`NotFound`, `Ambiguous`, `OutOfRange`), the whole batch aborts with **exit code 11** (`MultiEditOpFailed`) and nothing is written. The error names the op that failed:

```json
{
  "code": "MultiEditOpFailed",
  "opIndex": 2,
  "underlyingCode": "Ambiguous",
  "underlying": { "code": "Ambiguous", "data": { "needle": "...", "lineHits": [12, 47] } }
}
```

## Atomic multi-note changes: `jotura batch`

`jotura batch <FILE>` (or `-`) applies a JSON array of file-level operations across multiple notes. Use this when you want several notes updated together (e.g. "retag five notes and rename one folder") and want all-or-nothing semantics on the most common failure (hash conflict).

```sh
cat > /tmp/ops.json <<'JSON'
[
  {"path": "inbox/a.md", "op": {"kind": "write",      "content": "...", "ifHash": "..."}},
  {"path": "inbox/b.md", "op": {"kind": "edit",       "apply": [{"kind":"replace","old":"x","new":"X"}], "ifHash": "..."}},
  {"path": "inbox/c.md", "op": {"kind": "create",     "parent": "inbox", "name": "c.md"}},
  {"path": "old/d.md",   "op": {"kind": "rename",     "to": "archive/d.md"}},
  {"path": "trash/e.md", "op": {"kind": "delete"}},
  {"path": "inbox/f.md", "op": {"kind": "frontmatter-set", "key": "status", "value": "done"}},
  {"path": "inbox/g.md", "op": {"kind": "tag-add",    "tag": "project-x"}}
]
JSON

jotura batch /tmp/ops.json --diff
```

Always emits JSON to stdout (the global `--json` flag is implied):

```json
{
  "opsApplied": 7,
  "results": [
    {"path": "inbox/a.md", "kind": "write", "newHash": "...", "diff": "..."},
    {"path": "inbox/b.md", "kind": "edit",  "newHash": "...", "diff": "..."},
    ...
  ]
}
```

### Atomicity guarantee

1. **Plan phase**: the CLI reads current content for every path, runs body-modifying ops against it in-memory, and surfaces any `NotFound`/`Ambiguous`/`OutOfRange` errors **before** writing anything. Exit code `11` (`MultiEditOpFailed`) on failure; disk untouched.
2. **Precondition phase**: every `ifHash` is verified against current on-disk state in one pass. Any mismatch produces a `HashConflict` error with **all** conflicting paths and their per-path `currentVsIntended` diffs. Exit code `2`. Disk untouched.
3. **Apply phase**: the batch executes through the existing per-op core API, each op committing in its own SQLite transaction. The plan phase eliminates the common failure modes, but if an unrelated error (filesystem, manifest corruption, AlreadyExists) hits mid-batch, prior ops in that same batch have already committed and will NOT be rolled back.

In practice this means: **safe for "hash conflict" and "op planning errors" (the common cases). Best-effort for unrelated errors mid-write.** Always pass `ifHash` where the user might be editing concurrently.

### Batch op kinds

| `kind` | Required fields | Optional |
|---|---|---|
| `write` | `content` | `full`, `ifHash` |
| `edit` | `apply` (array of edit ops) | `ifHash` |
| `create` | `parent`, `name` | - |
| `rename` | `to` | - |
| `delete` | - | - |
| `frontmatter-set` | `key`, `value` | `ifHash` |
| `frontmatter-delete` | `key` | `ifHash` |
| `tag-add` | `tag` | `ifHash` |
| `tag-remove` | `tag` | `ifHash` |

`--dry-run` and `--diff` work on `batch` just like on single commands.
