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

`create` appends `.excalidraw.md` to a name with no board suffix and exits 9 if the file exists (use `add` then). Every other command takes the vault path or a bare name that resolves like a wikilink. `--spec -` reads the spec from stdin. Every mutating command takes `--if-hash`, `--dry-run`, `--diff` and `--json`; the JSON reply is `{path, hash, ids: {key: elementId}, added, changed, removed}`.

```sh
jotura canvas create <path> [--spec FILE|-] [--attach FILE]...
jotura canvas show <board>
jotura canvas add <board> --spec FILE|- [--attach FILE]...
jotura canvas update <board> --id ID [--text T] [--move X,Y] [--resize W,H] [style flags]
jotura canvas remove <board> --id ID...
jotura canvas layout <board> [--frame NAME] [--layout left-right|top-down|grid]
jotura canvas render <board> --out FILE.png|FILE.svg [--frame NAME] [--scale N]
```

## The spec

```json
{
  "frame": "Sync architecture",
  "layout": "left-right",
  "nodes": [ { "id": "app", "text": "Desktop app" } ],
  "edges": [ { "from": "app", "to": "api", "label": "push / pull" } ]
}
```

Board fields, all optional:

| Field | Values |
|---|---|
| `frame` | a name; wraps this spec's nodes in a named frame. Reusing the name of a frame already on the board adds to it and grows it |
| `layout` | `left-right` (default), `top-down`, `grid`, `none` |
| `nodes` | list of nodes |
| `edges` | list of edges |

`left-right` and `top-down` layer the nodes along the edges. `grid` fills rows for collections of cards. `none` puts nodes without coordinates in a row, in order. A node with both `x` and `y` is fixed in place by every layout.

### Nodes

Set exactly one content field per node.

| Field | Becomes |
|---|---|
| `text` | a box with the text as its label; `shape` picks the box |
| `text` with `shape: "text"` | free text with no box |
| `note` | a card linking a note, written as one `[[Note name]]` |
| `image` | an image: a vault path, or the name of a file passed with `--attach FILE` (imported into `assets/`) |
| `draw` | a freehand stroke from a list of `[x, y]` points, at least 2; optional `pressure` list of the same length |
| `line` | a free line from at least 2 points |
| `arrow` | an arrow with no bindings, from at least 2 points |

Other node fields: `id` (your key, unique in the spec), `shape` (`rect` default, `ellipse`, `diamond`, `text`; only with `text`), `x` and `y` (explicit position), `width` and `height` (override the computed size; an image keeps its aspect ratio when only `width` is given).

Sizes when you give none: boxes fit their label (at least 160 by 80, and a label wider than 320 px on one line wraps to about 280 px first), cards are 360 by 240, images are natural size capped at 480 wide.

### Style

Any node accepts these. Defaults match Excalidraw's.

| Field | Values |
|---|---|
| `stroke` | palette name (uses its dark shade) or `#rrggbb` |
| `fill` | palette name (uses its light shade) or `#rrggbb` |
| `fillStyle` | `hachure`, `cross-hatch`, `solid` |
| `strokeWidth` | `thin`, `bold`, `extra-bold`, or a number |
| `strokeStyle` | `solid`, `dashed`, `dotted` |
| `roughness` | `architect`, `artist`, `cartoonist`, or 0 to 2 |
| `opacity` | 0 to 100 |
| `font` | `hand`, `normal`, `code` |
| `fontSize` | `s`, `m`, `l`, `xl`, or a number |

Palette names: `gray`, `red`, `pink`, `grape`, `violet`, `blue`, `cyan`, `teal`, `green`, `yellow`, `orange`, `bronze`, `black`, `white`, `transparent`. A `fill` needs `fillStyle: "solid"` to show as a flat colour.

### Edges

| Field | Values |
|---|---|
| `from`, `to` | a node id in this spec, or the key or element id of something already on the board |
| `label` | text on the arrow |
| `style` | `solid`, `dashed`, `dotted` |
| `stroke` | a colour, as above |
| `startHead`, `endHead` | `none`, `arrow`, `bar`, `dot`, `triangle`; the end defaults to `arrow`, the start to `none` |

An edge has no id in a spec. The arrows bend around boxes and labels without help, and two edges between the same pair are drawn side by side. A self-edge (`from` equal to `to`) is refused; add an `arrow` node with points instead.

## Keys

Each spec `id` becomes that element's key. Later commands take a key, an element id, or for a frame its name. A box's label is part of the box: a label's id resolves to its box, so `remove <label id>` removes the box. Removing a node also removes its arrows. Removing a frame keeps its children and they leave the frame.

`update` changes one element at a time and takes `--text`, `--move X,Y`, `--resize W,H` and a flag per style field: `--stroke`, `--fill`, `--fill-style`, `--stroke-width`, `--stroke-style`, `--roughness`, `--opacity`, `--font`, `--font-size`. Moving or resizing a box takes its label and its arrows along. An edge cannot be updated directly; change the boxes it joins.

## Reading a board

`show` always prints JSON, with the board's `path` and `hash`:

```json
{
  "path": "Projects/Sync architecture.excalidraw.md",
  "hash": "9f2c...",
  "nodes": [
    { "id": "k3f9a1bc", "key": "app", "kind": "box", "shape": "rect", "text": "Desktop app",
      "x": 0, "y": 0, "w": 200, "h": 80, "frame": "Sync architecture",
      "style": { "stroke": "#1e1e1e", "fill": "transparent", "fillStyle": "solid",
                 "strokeWidth": 2, "strokeStyle": "solid", "roughness": 1, "opacity": 100,
                 "font": "hand", "fontSize": 20 },
      "by": "agent" }
  ],
  "edges": [
    { "id": "Qp0xYt7a", "key": null, "from": "app", "to": "api", "label": "push / pull", "style": { "stroke": "#1e1e1e", "strokeStyle": "solid" } }
  ]
}
```

`kind` is `box`, `text`, `note`, `image`, `draw`, `line`, `arrow` or `frame`. `note` holds the `[[link]]` of a card and `image` the file of an image. Labels are folded into their box, so they are not separate nodes. An edge's `style` carries the same fields as a node's, and `show` names the arrow by its element id, which `remove --id` accepts. `from` and `to` use a key when the element has one and an element id otherwise. `by` says who made the element: `agent` for what the CLI created, `user` for what a person drew. Run `show` before touching a board you did not just build, and again whenever the user may have changed it.

## Looking after the user's drawing

The CLI never moves or changes anything a person drew. `canvas add` places new nodes in free space, beside the element the first new edge connects to, or to the right of everything else when nothing connects. `canvas layout` re-lays out every element the CLI created, including ones you placed with explicit `x` and `y`, and leaves the user's elements where they are. `canvas update` on a user's element changes it as asked and the element stays theirs.

## Examples

**Architecture diagram with a frame and labelled edges.**

```sh
jotura canvas create "Projects/Sync architecture" --spec - --json <<'EOF'
{
  "frame": "Sync architecture",
  "layout": "left-right",
  "nodes": [
    { "id": "app", "text": "Desktop app" },
    { "id": "api", "text": "Workers API", "fill": "blue", "fillStyle": "solid" },
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
  "nodes": [ { "id": "risk", "text": "Risk: store review", "fill": "yellow", "fillStyle": "solid" } ],
  "edges": [ { "from": "risk", "to": "Zk39sLq2", "style": "dashed" } ]
}
EOF
jotura canvas render "Whiteboard/Launch plan" --out /tmp/plan.png
```

The new box lands beside "Launch" and the person's boxes stay put. To retitle your box later: `jotura canvas update "Whiteboard/Launch plan" --id risk --text "Risk: review delay"`.

## Images

`image` takes a path to a file already in the vault, or the name of a file you pass with `--attach`:

```sh
jotura canvas add "Whiteboard/Launch plan" --attach ~/Downloads/dashboard.png --spec - <<'EOF'
{ "nodes": [ { "id": "shot", "image": "dashboard.png", "width": 320 } ] }
EOF
```

The attached file is imported into `assets/` and referenced from the board.

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 2 | `HashConflict`, the board changed since your `--if-hash` | run `show` for the new hash and state, re-apply your change |
| 3 | `NotFound`, a key, element id, frame name, image file or board is missing | run `show` and use a key or id it lists; check the board path |
| 8 | `PermissionDenied` | tell the user |
| 9 | `InvalidArgs`, a bad spec or flag; the message names the field and lists the valid values | fix the field it names and retry; nothing was written |
| 15 | `ReadOnlyShare`, the board is in a view-only share | stop and ask the owner for editor access |
