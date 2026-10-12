# Canvas boards

A board is an Excalidraw whiteboard stored in the vault as `Name.excalidraw.md` (the default) or `Name.excalidraw` (plain JSON). The user opens it as an endless canvas in the desktop app, and Obsidian opens it too. You never write Excalidraw JSON. You describe the board in a small spec, `jotura canvas` expands it into elements, lays it out and writes it atomically, and you render it to a PNG to check your work.

Make a board when the content is spatial: an architecture or flow diagram, a grid of note cards, a sketch with labelled arrows, a collection of images. Use a plain note for prose, lists and anything the user will mostly read top to bottom.

**Never edit the `## Drawing` block by hand**, and never `edit`, `write` or `batch` a board. The block holds element ids, seeds, versions and bindings that must agree with each other, and a hand edit that breaks one loses the user's drawing. Every change goes through `jotura canvas`.

## The workflow

```sh
# 1. Create the board from a spec
jotura canvas create "Projects/Sync architecture" --spec spec.json --json

# 2. Render it and look at it (open the PNG with your image tool, see references/images.md)
jotura canvas render "Projects/Sync architecture" --out /tmp/sync.png

# 3. Change what is wrong, by key
jotura canvas update "Projects/Sync architecture" --id api --text "Workers API" --fill green
jotura canvas add "Projects/Sync architecture" --spec more.json
jotura canvas remove "Projects/Sync architecture" --id r2

# 4. Render again and confirm
jotura canvas render "Projects/Sync architecture" --out /tmp/sync.png
```

Always render and look before you report the board as done. Render output can differ slightly from the app in line breaks and stroke wobble, so check structure and spacing and ignore pixel detail.

```sh
jotura canvas create <path> [--spec FILE|-] [--attach FILE]...
jotura canvas show <board>
jotura canvas add <board> --spec FILE|- [--attach FILE]...
jotura canvas update <board> --id ID [--text T] [--move X,Y] [--resize W,H] [style flags]
jotura canvas remove <board> --id ID...
jotura canvas layout <board> [--frame NAME] [--layout left-right|top-down|grid]
jotura canvas render <board> --out FILE.png|FILE.svg [--frame NAME] [--scale N]
```

`<board>` is a vault path or a bare name that matches one board's file name. Several boards with that name exit 4 and list `candidates`; pass the path then. `create` appends `.excalidraw.md` to a name with no board suffix, exits 9 if the file exists (use `add` then) and exits 9 on `--if-hash`, since a new board has no hash. `--spec -` reads the spec from stdin.

Every command that changes a board takes `--if-hash HASH`, `--dry-run`, `--diff` and `--json`. The JSON reply is `{path, hash, ids: {key: elementId}, added, changed, removed}`, plus `created: true` on `create` and `diff` with `--diff`. The counts are elements, so a box and its label count as two. With `--dry-run` nothing is written and the reply is `{dryRun: true, path, currentHash, newHash, changes, preview: {kind, added, changed, removed, ids}}`.

`render` changes nothing in the vault. `--out` must end in `.svg` or `.png` (the extension picks the format) and can be any path on disk. `--frame` draws one frame and its children. `--scale` is the PNG pixel scale: default 2, above 0 and at most 8, and a very large board is drawn at a lower scale. The reply is `{path, out, format, width, height, scale, scaleReduced}`, where `scale` is the scale used and `scaleReduced` reports that the board forced it below the scale you asked for. An unknown frame exits 3 and the message lists the board's frames.

## The spec

```json
{
  "frame": "Sync architecture",
  "layout": "left-right",
  "nodes": [ { "id": "app", "text": "Desktop app" } ],
  "edges": [ { "from": "app", "to": "api", "label": "push / pull" } ]
}
```

Board fields, all optional. An unknown field anywhere in the spec exits 9 and the message lists the fields that exist.

| Field | Values |
|---|---|
| `frame` | a name; wraps this spec's nodes and edges in a frame of that name. A frame the CLI made earlier with the same name is reused and grown. A frame a person drew is never reused, so a second frame is made |
| `layout` | `left-right` (default), `top-down`, `grid`, `none` |
| `nodes` | list of nodes |
| `edges` | list of edges |

`left-right` and `top-down` layer the nodes along the edges. `grid` fills rows for collections of cards. `none` puts nodes without coordinates in a row, in order. A node with both `x` and `y` keeps that position in the spec's layout.

### Nodes

Set exactly one content field per node.

| Field | Becomes |
|---|---|
| `text` | a box with the text as its label; `shape` picks the box |
| `text` with `shape: "text"` | free text with no box |
| `note` | a card linking a note, written as one `[[Note name]]` |
| `image` | an image: a vault path or file name (resolved like a wikilink), or the file name of a file passed with `--attach FILE`. PNG, JPEG, GIF, WebP and SVG |
| `draw` | a freehand stroke from a list of `[x, y]` points, at least 2 and not all the same; optional `pressure` list with one number per point |
| `line` | a free line from at least 2 points |
| `arrow` | an arrow bound to nothing, from at least 2 points |

Other node fields: `id` (your key, unique in the spec), `shape` (`rect` default, `ellipse`, `diamond`, `text`; only with `text`), `x` and `y` (pin the top-left corner when both are given; for `draw`, `line` and `arrow` they place the point list), `width` and `height` (positive numbers that override the computed size; an image keeps its aspect ratio when only one is given, and free text wraps to `width`).

Sizes when you give none: boxes fit their label (label plus 40 px, at least 160 by 80, and a label wider than 320 px on one line wraps to about 280 px first), cards are 360 by 240, images are natural size capped at 480 wide.

### Style

Any node accepts these. The defaults are Excalidraw's.

| Field | Values | Default |
|---|---|---|
| `stroke` | palette name (its dark shade) or `#rgb` / `#rrggbb` | `black` |
| `fill` | palette name (its light shade) or `#rgb` / `#rrggbb` | `transparent` |
| `fillStyle` | `hachure`, `cross-hatch`, `solid` | `solid` |
| `strokeWidth` | `thin` (1), `bold` (2), `extra-bold` (4), or a positive number | `bold` |
| `strokeStyle` | `solid`, `dashed`, `dotted` | `solid` |
| `roughness` | `architect` (0), `artist` (1), `cartoonist` (2), or a number from 0 to 2 | `artist` |
| `opacity` | 0 to 100 | 100 |
| `font` | `hand`, `normal`, `code` | `hand` |
| `fontSize` | `s` (16), `m` (20), `l` (28), `xl` (36), or a positive number | `m` |

Palette names: `gray`, `red`, `pink`, `grape`, `violet`, `blue`, `cyan`, `teal`, `green`, `yellow`, `orange`, `bronze`, `black`, `white`, `transparent`. Names are case-insensitive. Because `fillStyle` defaults to `solid`, a `fill` shows as a flat colour without further fields. On a box, `stroke` and `fill` style the shape, and `font` and `fontSize` style its label.

### Edges

| Field | Values |
|---|---|
| `from`, `to` | required: a node id in this spec, or the key or element id of something already on the board |
| `label` | text on the arrow |
| `style` | `solid` (default), `dashed`, `dotted` |
| `stroke` | a colour, as above; a palette name uses its dark shade |
| `startHead`, `endHead` | `none`, `arrow`, `bar`, `dot`, `triangle`; the end defaults to `arrow`, the start to `none` |

An edge has no key. Only boxes, free text, notes, images and frames can be connected. A self-edge (`from` equal to `to`) is refused; add an `arrow` node with points instead. An edge that names a node that does not exist exits 3. The arrows bend around boxes and labels without help.

## Keys

Each spec `id` becomes that element's key. Later commands take a key, an element id, or for a frame its name. `add` refuses an `id` that is already a key on the board. A box's label is part of the box: a label's id resolves to its box, so `remove <label id>` removes the box. Removing a node also removes its labels and arrows. Removing a frame keeps its children and they leave the frame. An edge has no key, so name it by the element id that `show` lists: `remove --id <arrow id>` deletes that edge and its label.

`update` changes one element at a time and takes `--text`, `--move X,Y`, `--resize W,H` and a flag per style field: `--stroke`, `--fill`, `--fill-style`, `--stroke-width`, `--stroke-style`, `--roughness`, `--opacity`, `--font`, `--font-size`. It exits 9 when you pass none of them. A box grows to fit new text and never shrinks. Moving or resizing a box takes its label along and re-routes its arrows, and moving a frame moves its children. `--text` works on boxes, free text, frames and edges (an edge without a label gets one); on a note, image, line or freehand stroke it exits 9. `--resize` exits 9 on `draw`, `line` and `arrow` elements. An edge accepts `--text`, `--stroke`, `--stroke-width`, `--stroke-style`, `--opacity`, `--roughness`, `--font` and `--font-size`, and refuses `--move`, `--resize`, `--fill` and `--fill-style` with exit 9: edges follow the boxes they join, so move a box instead.

## Reading a board

`show` always prints JSON, with the board's `path` and `hash`:

```json
{
  "path": "Projects/Sync architecture.excalidraw.md",
  "hash": "9f2c...",
  "nodes": [
    { "id": "k3f9a1bc", "key": "app", "kind": "box", "shape": "rect", "text": "Desktop app",
      "x": 0.0, "y": 0.0, "w": 162.7, "h": 80.0, "frame": "Sync architecture",
      "style": { "stroke": "black", "fill": "transparent", "fillStyle": "solid",
                 "strokeWidth": "bold", "strokeStyle": "solid", "roughness": "artist", "opacity": 100,
                 "font": "hand", "fontSize": "m" },
      "by": "agent" }
  ],
  "edges": [
    { "id": "Qp0xYt7a", "from": "app", "to": "api", "label": "push / pull",
      "style": { "stroke": "black", "fill": "transparent", "fillStyle": "solid",
                 "strokeWidth": "bold", "strokeStyle": "solid", "roughness": "artist", "opacity": 100,
                 "font": "hand", "fontSize": "m" } }
  ]
}
```

`kind` is `box`, `text`, `note`, `image`, `draw`, `line`, `arrow` or `frame`. `note` holds the `[[link]]` of a card, `image` the file of an image, and a frame's `text` is its name. `key` is left out for an element that has none, and so are `shape`, `text`, `note`, `image` and `frame` when they do not apply. Labels are folded into their box or arrow, so they are not separate nodes. An edge's `style` has the same fields as a node's (`font` and `fontSize` only when it has a label). Colours, widths, roughness and font sizes come back as the names in the style table when one matches, and as raw values otherwise. `from` and `to` use a key when the element has one and an element id otherwise. `by` is `agent` for what the CLI created and `user` for what a person drew. Run `show` before touching a board you did not just build, and again whenever the user may have changed it.

## Looking after the user's drawing

The CLI never moves or changes anything a person drew unless a command targets it. `canvas add` places new nodes in free space, to the right of the element the first new edge connects to, or to the right of everything else when nothing connects. An edge from a new node to a person's element adds the arrow's binding to that element and leaves its position alone. `canvas layout` re-lays out every element the CLI created, including ones you pinned with `x` and `y`, and leaves the user's elements where they are. `canvas update` and `canvas remove` act on any element you name, and an element a person drew stays theirs afterwards.

## Examples

**Architecture diagram with a frame and labelled edges.**

```sh
jotura canvas create "Projects/Sync architecture" --spec - --json <<'EOF'
{
  "frame": "Sync architecture",
  "layout": "left-right",
  "nodes": [
    { "id": "app", "text": "Desktop app" },
    { "id": "api", "text": "Workers API", "fill": "blue" },
    { "id": "r2",  "text": "R2 blobs", "shape": "ellipse" },
    { "id": "db",  "text": "D1", "shape": "diamond", "strokeStyle": "dashed" }
  ],
  "edges": [
    { "from": "app", "to": "api", "label": "push / pull" },
    { "from": "api", "to": "r2",  "label": "sealed blobs" },
    { "from": "api", "to": "db",  "style": "dashed", "endHead": "triangle" }
  ]
}
EOF
jotura canvas render "Projects/Sync architecture" --out /tmp/sync.png
```

**A grid of note cards.** Each card links one note.

```sh
jotura canvas create "Reading/Sync notes" --spec - <<'EOF'
{
  "layout": "grid",
  "nodes": [
    { "id": "n1", "note": "[[Sync engine]]" },
    { "id": "n2", "note": "[[Key rotation]]" },
    { "id": "n3", "note": "[[Share trust]]" },
    { "id": "n4", "note": "[[Embedding cache]]" }
  ]
}
EOF
```

**Adding to a board a person already drew.** Read it first. The user's elements have no key, so connect to them by element id.

```sh
jotura canvas show "Whiteboard/Launch plan" | jq '.nodes[] | {id, text, by}'
# { "id": "Zk39sLq2", "text": "Launch", "by": "user" }

jotura canvas add "Whiteboard/Launch plan" --spec - --json <<'EOF'
{
  "nodes": [ { "id": "risk", "text": "Risk: store review", "fill": "yellow" } ],
  "edges": [ { "from": "risk", "to": "Zk39sLq2", "style": "dashed" } ]
}
EOF
jotura canvas render "Whiteboard/Launch plan" --out /tmp/plan.png
```

The new box lands to the right of "Launch" and the person's boxes stay put. To retitle your box later: `jotura canvas update "Whiteboard/Launch plan" --id risk --text "Risk: review delay"`.

## Images

`image` takes the name of a file you pass with `--attach`, or a path or file name of an image already in the vault (resolved like a wikilink):

```sh
jotura canvas add "Whiteboard/Launch plan" --attach ~/Downloads/dashboard.png --spec - <<'EOF'
{ "nodes": [ { "id": "shot", "image": "dashboard.png", "width": 320 } ] }
EOF
```

On a `.excalidraw.md` board the attached file is copied into `assets/` and linked from the board. A plain `.excalidraw` board embeds the bytes. A name that is neither attached nor in the vault exits 3.

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 2 | `HashConflict`, the board changed since your `--if-hash`; the error carries a `currentVsIntended` diff | run `show` for the new hash and state, re-apply your change |
| 3 | `NotFound`: the board, a key, element id or frame name, the spec file, an `--attach` file, a vault image, or an edge endpoint does not exist | run `show` and use a key or id it lists; check the path |
| 4 | `Ambiguous`, a bare board name matches several boards | pass the vault path from `candidates` |
| 8 | `PermissionDenied` | tell the user |
| 9 | `InvalidArgs`: a bad spec or flag, a board that already exists on `create`, `--if-hash` on `create`, a file that is not a board, an `update` with nothing to change, or an edge asked to move or take a fill; the message names the field and lists the valid values | fix the field it names and retry; nothing was written |
| 14 | `TargetMissing`, the board was deleted or moved before the write | check the board still exists, then recreate or retry |
| 15 | `ReadOnlyShare`, the board is in a view-only share | stop and ask the owner for editor access |
