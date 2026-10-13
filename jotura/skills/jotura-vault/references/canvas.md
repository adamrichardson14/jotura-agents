# Canvases

A canvas is an Excalidraw drawing stored in the vault as `Name.excalidraw.md` (the default) or `Name.excalidraw` (plain JSON). The user opens it as an endless canvas in the desktop app, and Obsidian opens it too. You describe the canvas in a small spec and `jotura canvas` writes the Excalidraw elements for you, with the layout and the arrow routing done. You never write Excalidraw JSON.

Make a canvas when the content is spatial: an architecture diagram, a flow, note cards in a grid, a sketch with labelled arrows. Use a plain note for text the user will read from top to bottom.

**Never edit the `## Drawing` block by hand**, and never `edit`, `write` or `batch` a canvas file. The block holds element ids, seeds, versions and bindings that must agree with each other, and a hand edit that breaks one loses the user's drawing. Every change goes through `jotura canvas`.

## The workflow

```sh
# 1. Create the canvas from a spec
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

Render and look before you report the canvas as done. A render differs slightly from the app in line breaks and stroke wobble, so judge structure and spacing from it.

On a canvas you did not build, run the change with `--dry-run --diff` first and read the diff before you write it. To see where new nodes will land, run `add --dry-run --json` and read `preview.rects`; they match the real write. A dry run's `newHash` differs from the hash of the real write, because new ids and seeds are random. Take the hash for `--if-hash` from `show` or from the reply of a real write.

`--vault <path>` is a global flag on every command, and `JOTURA_VAULT` sets it for the shell. Windows paths work with forward slashes, as in `--vault C:/Users/me/vault`. A relative `--out` on `render` is relative to the current directory.

```sh
jotura canvas create <path> [--spec FILE|-] [--attach FILE]...
jotura canvas show <canvas>
jotura canvas add <canvas> --spec FILE|- [--attach FILE]...
jotura canvas update <canvas> --id ID [--text T] [--move X,Y] [--resize W,H] [style flags]
jotura canvas remove <canvas> --id ID...
jotura canvas layout <canvas> [--frame NAME] [--layout left-right|top-down|grid]
jotura canvas render <canvas> --out FILE.png|FILE.svg [--frame NAME] [--scale N]
```

`<canvas>` is a vault path (the canvas suffix is optional, so `Projects/Sync` finds `Projects/Sync.excalidraw.md`, even when a folder `Projects/Sync/` exists) or a bare name that matches one canvas's file name. Several canvases with that name exit 4 and list `candidates`; pass the path then. `create` appends `.excalidraw.md` to a name with no canvas suffix, exits 9 if the file exists (use `add` then) and exits 9 on `--if-hash`, since a new canvas has no hash. `--spec -` reads the spec from stdin.

To replace a canvas wholesale, move it to the vault trash and create it again. `trash` takes the canvas file path with its suffix, and a bare canvas name exits 3. `jotura trash restore "Projects/Sync.excalidraw.md"` brings the old canvas back. Use `trash`, never `jotura delete`, which removes the file for good.

```sh
jotura trash "Projects/Sync.excalidraw.md"
jotura canvas create "Projects/Sync" --spec spec.json --json
```

Every command that changes a canvas takes `--if-hash HASH`, `--dry-run`, `--diff` and `--json`. The JSON reply is `{path, hash, ids: {key: elementId}, added, changed, removed}`, plus `created: true` on `create` and `diff` with `--diff`. On `create` and `add` it also carries `rects`, the placed rectangle of each new keyed node as `{x, y, width, height}`, keyed like `ids`; read it to check where nodes landed without a render. The counts are elements, so a box and its label count as two. `layout` takes no key, so its `ids` lists the keyed elements that moved. With `--dry-run` nothing is written and the reply is `{dryRun: true, path, currentHash, newHash, changes, preview: {kind, added, changed, removed, ids, rects}}`, with `rects` on `create` and `add` only.

Without `--json`, these commands print one `key -> id` line per entry in `ids`, or `path: N added, N changed, N removed` when `ids` is empty.

`render` writes only the output file. `--out` must end in `.svg` or `.png` (the extension picks the format); a relative path is relative to the current directory. An `--out` inside the vault is queued for sync, and it exits 15 in a view-only share. When `--out` names an existing file inside the vault, the old file moves to the vault trash first and `jotura trash restore` brings it back. An `--out` that would overwrite an image the canvas uses exits 9. `--frame` draws one frame and its children. `--scale` applies to PNG only: default 2, above 0 and at most 8, and a canvas too large for the requested scale is drawn at a lower one.

The JSON reply is `{path, out, format, width, height}` plus these fields when they apply: `scale` (PNG only, the scale used), `scaleReduced: true` (only when the scale was lowered) and `missingImages` (only when images could not be loaded; vault paths or file ids, drawn as placeholders) and `replacedToTrash` (only when an existing vault file was moved to the trash). Plain-text mode prints `rendered OUT (WxH)`, and a missing image adds `warning: N image(s) not found, drawn as placeholders: ...` on stderr. An unknown frame exits 3 and the message lists every frame, with `(unnamed) <id>` for an unnamed one and `name (id)` when two share a name.

## The spec

```json
{
  "frame": "Sync architecture",
  "layout": "left-right",
  "nodes": [ { "id": "app", "text": "Desktop app" } ],
  "edges": [ { "from": "app", "to": "api", "label": "push / pull" } ]
}
```

Top-level spec fields, all optional. An unknown field anywhere in the spec exits 9 and the message lists the fields that exist.

| Field | Values |
|---|---|
| `frame` | a name; every node without its own `frame` goes in a frame of that name, with the edges between those nodes, and the frame grows to hold them. A frame the CLI made earlier with the same name is reused and grown. A frame a person drew is left alone and a second frame is made. For several frames, set `frame` per node (see "Several frames") |
| `layout` | `left-right` (default), `top-down`, `grid`, `none` |
| `columns` | a whole number from 1 to 20, for `layout: "grid"` only. The default is the number of nodes for up to 3 nodes, and otherwise the square root of the node count rounded up |
| `nodes` | list of nodes |
| `edges` | list of edges |

`left-right` and `top-down` layer the nodes along the edges, with unconnected groups side by side. A node with several parents sits centred between them, and a parent with several children sits centred over them. A group of 3 to 8 nodes joined in one simple cycle (each node has one edge in and one edge out, and the group has no other edges) is drawn as a ring with straight arrows, clockwise in edge order, starting from the top-left for `left-right` and from the top for `top-down`. The gap between layers widens to fit edge labels, with room for the arrow on both sides of a label. `grid` fills rows and suits cards. In a grid every text box takes the size of the largest one in its frame, so a matrix lines up; a node that sets `width` or `height` keeps its own size, and cards and images keep theirs. `none` puts nodes without coordinates in a row, in order. `none` exists in the spec only: `canvas layout --layout none` exits 9. A node with both `x` and `y` keeps that position in the spec's layout.

### Nodes

Set exactly one content field per node.

| Field | Becomes |
|---|---|
| `text` | a box with the text as its label; `shape` picks the box |
| `text` with `shape: "text"` | free text with no box |
| `note` | a card linking a note, written as one `[[Note name]]`. A render shows the note's title and a "Note" caption; the desktop app shows a linked card for the note |
| `image` | an image: a vault path or file name (resolved like a wikilink), or the file name of a file passed with `--attach FILE`. PNG, JPEG, GIF, WebP and SVG. An image has no border: put it in a frame or add a box node behind it. `--stroke`, `--stroke-style` and `--stroke-width` on an image exit 9 |
| `draw` | a freehand stroke from a list of `[x, y]` points, at least 2 and not all the same; optional `pressure` list with one number per point |
| `line` | a free line from at least 2 points |
| `arrow` | an arrow bound to nothing, from at least 2 points |

Other node fields: `id` (your key, unique in the spec), `frame` (the frame this node goes in; it overrides the top-level `frame`), `shape` (`rect` default, `ellipse`, `diamond`, `text`; only with `text`), `x` and `y` (pin the top-left corner when both are given; for `draw`, `line` and `arrow` they place the point list), `width` and `height` (positive numbers that override the computed size; an image keeps its aspect ratio when only one is given, and free text wraps to `width`).

Write `\n` in `text` to start a new line in the label: `"CEO\nMarcus Hale"` is a two-line box.

Sizes when you give none: a box is its label plus 40 px, at least 160 by 80, and a one-line label wider than 320 px wraps to about 280 px first. Set `width` to fix a box's width: the label wraps inside it and the box grows taller to fit. Give a row of related boxes the same `width` when they should line up. A box or diamond grows to fit its label, including after `update --text`. A label outside the Latin alphabet is sized approximately, and PNG renders draw it in a fallback face. A card is 360 by 240. An image keeps its natural size up to 480 px wide, and `width` alone scales the height to keep the aspect ratio.

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

Palette names: `gray`, `red`, `pink`, `grape`, `violet`, `blue`, `cyan`, `teal`, `green`, `yellow`, `orange`, `bronze`, `black`, `white`, `transparent`. Names are case-insensitive. `fillStyle` defaults to `solid`, so a `fill` shows as a flat colour with no further fields. On a box, `stroke` and `fill` style the shape, and `font` and `fontSize` style its label. `fill` colours the inside of a shape with the light shade. `stroke` colours the outline and the text with the dark shade. A request to "colour it blue" usually means `fill`.

### Edges

| Field | Values |
|---|---|
| `from`, `to` | required: a node id in this spec, or the key or element id of something already on the canvas. A frame name does not resolve here |
| `label` | text on the arrow |
| `style` | `solid` (default), `dashed`, `dotted` |
| `stroke` | a colour, as above; a palette name uses its dark shade |
| `startHead`, `endHead` | `none`, `arrow`, `bar`, `dot`, `triangle`; the end defaults to `arrow`, the start to `none` |

The arrow points from `from` to `to`. For a relation such as "connected to", pick the direction the data or the action flows. An edge has no key. Only boxes, free text, notes, images and frames can be connected. A self-edge (`from` equal to `to`) is refused; add an `arrow` node with points instead. Two ends that overlap are refused with "overlap, so the arrow has no length" (exit 9); move one of them. An edge that names a node that does not exist exits 3. The arrows keep clear of boxes without help, and edge labels stay 30 px apart whenever a route allows it. Cycles and loop-back edges are routed automatically. The layout reserves a lane for a loop-back edge between the layers it crosses, and the arrow takes the shortest clear path back: through that lane, as a bend beside its forward partner, or round the nearest side of the two nodes it joins. You cannot choose which side an arrow uses. An arrow with more than one bend is drawn with sharp corners, so its label sits on the line. Top-down layers leave about 55 px of arrow on each side of a label. Keep edge labels to a few words (about 20 characters); a longer label wraps onto two lines, as in the desktop app.

### Several frames

Build a canvas with several frames in one `create`, and give each node its own `frame`. Adding the frames one `add` at a time places every new frame beside the last one, so the canvas comes out as a staircase instead of a flow.

```sh
jotura canvas create "Projects/Stack" --spec - --json <<'EOF'
{ "layout": "left-right",
  "nodes": [
    { "id": "desktop", "text": "Desktop app", "frame": "Clients" },
    { "id": "phone", "text": "Phone app", "frame": "Clients" },
    { "id": "gw", "text": "API gateway", "frame": "Edge" },
    { "id": "pg", "text": "Postgres", "frame": "Storage" } ],
  "edges": [
    { "from": "desktop", "to": "gw" }, { "from": "phone", "to": "gw" },
    { "from": "gw", "to": "pg" } ] }
EOF
```

Each frame's nodes are laid out on their own, using the edges inside the frame. A frame with no inner edges stacks its nodes across the flow: a column for `left-right`, a row for `top-down`. Each frame is then placed as one block, with the same layered layout over the edges between frames, so the frames follow the flow, and the nodes inside a frame are ordered to face the frames they connect to. Nodes without a frame take part as blocks of their own. Edges between frames are routed like any other edge.

| `layout` | Inside each frame | The frames |
|---|---|---|
| `left-right` | layered along the inner edges; a column when there are none | follow the flow to the right |
| `top-down` | layered along the inner edges; a row when there are none | follow the flow downwards |
| `grid` | a grid of `columns` | laid out in a grid |
| `none` | a column | one row |

In a spec with several frames, pin every node of a frame or none of them. A frame that mixes pinned and laid-out nodes exits 9 and the message names the frame. A frame whose nodes are all pinned stays where you put it, and the laid-out frames move to its right when they would overlap it.

## Placement, spacing and titles

- `add` puts the new block 120 px to the right of the element its first new edge connects to, centred on it. When that spot is taken it steps down 40 px at a time to the next free spot. With no connecting edge the block goes to the right of everything on the canvas. `add` leaves 60 px between a new node and the boxes, text and frames already there, and 20 px between it and existing arrows.
- When the spec's `frame` names a frame the CLI made earlier, `add` places the block inside that frame by the same rule: the frame itself does not count as taken, and it grows to hold the block. A `frame` that makes a new frame keeps the block clear of everything by the frame's padding and title as well.
- When an `add` spec puts nodes in several frames, each frame is placed as its own block in turn: frames the CLI made earlier first, then nodes without a frame, then new frames, so a new frame never lands inside an existing one.
- Let the layout place nodes. Pin `x` and `y` only when the picture needs an arrangement the layout cannot express, such as a map, a floor plan, a timeline at fixed dates or a title above the canvas. Pinning every node turns the layout off and leaves the arrow routing to work around your coordinates.
- To change spacing, pin `x` and `y` on nodes in the spec, or run `update --move X,Y` afterwards. Moving a node inside a frame the CLI made grows the frame to hold it. Pinned nodes never take a layout slot, so the laid-out nodes still start at (0,0).
- Grid cells start at 0,0, and a card cell is 360 by 240 with 40 px gaps. For a title above a canvas or a grid, add a free text node with a pinned position above the top-left of the laid-out nodes: `{ "text": "Reading list", "shape": "text", "fontSize": "xl", "x": 0, "y": -80 }`. The title sits at y -80 above nodes that start at (0,0). Free text is 1.25 times its font size tall per line in the hand font, so a `m` (20 px) line is 25 px high. A frame name works as a small grey label when a title does not need to stand out.
- To put a frame around a single node, for example to give an image a visible border, create the other nodes first and then `canvas add` that node with `frame` set. A frame pads its children by 40 px, and its name shows as a small grey label above it.
- To make cards for the notes in a folder, list them with `jotura ls --folder <folder> --json` and put one `note` node per path in the spec.

## Keys

- Each spec `id` becomes that element's key. Later commands take a key, an element id or, for `update`, `remove` and `layout`, a frame name.
- `add` refuses an `id` that is already a key on the canvas. Pick another id, or use `update`.
- A box's label is part of the box. A label's id resolves to its box, so `remove <label id>` removes the box.
- Removing a box removes its label and the arrows the CLI made to it, with their labels. An arrow a person drew stays on the canvas and only loses its binding to the removed box. Removing a frame keeps its children and takes them out of the frame.
- An edge has no key. Name it by the element id that `show` lists: `remove --id <arrow id>` deletes that edge and its label.

## Changing one element

`update` changes one element at a time. It takes `--text`, `--move X,Y`, `--resize W,H` and a flag per style field: `--stroke`, `--fill`, `--fill-style`, `--stroke-width`, `--stroke-style`, `--roughness`, `--opacity`, `--font`, `--font-size`. With none of them it exits 9.

- A box grows to fit new text and never shrinks.
- Moving or resizing a box takes its label along and re-routes its arrows. Moving a frame moves its children.
- `--text` works on boxes, free text, frames, edges and free `arrow` nodes; an edge or arrow without a label gets one. On a note, image, line or freehand stroke it exits 9.
- `--resize` exits 9 on `draw`, `line` and `arrow` elements.
- An edge accepts `--text`, `--stroke`, `--stroke-width`, `--stroke-style`, `--opacity`, `--roughness`, `--font` and `--font-size`. It refuses `--move`, `--resize`, `--fill` and `--fill-style` with exit 9, because edges follow the boxes they join. Move a box instead.

`layout` re-arranges boxes, notes, images and free text that the CLI created, on the whole canvas or in one frame (`--frame` takes a frame name or id). Freehand strokes, lines, free arrows and frames stay where they are, and so does everything a person drew. It does move nodes you pinned with `x` and `y`. Arrows are routed again afterwards, and frames the CLI made are resized to fit. On a canvas with several CLI frames, `layout` lays all the frames out together, the same way `create` does with several frames. `--layout` takes `left-right` (the default), `top-down` or `grid`; `none` exits 9.

## Reading a canvas

`show` always prints JSON, with the canvas's `path` and `hash`:

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
      "style": { "stroke": "black", "strokeWidth": "bold", "strokeStyle": "solid",
                 "roughness": "artist", "opacity": 100, "font": "hand", "fontSize": "m" } }
  ]
}
```

`kind` is `box`, `text`, `note`, `image`, `draw`, `line`, `arrow` or `frame`. `note` holds the `[[link]]` of a card, `image` the file of an image, and a frame's `text` is its name. `key` is left out for an element that has none, and so are `shape`, `text`, `note`, `image` and `frame` when they do not apply. Labels are folded into their box or arrow, so they are not separate nodes. An edge's `style` has the node fields minus `fill` and `fillStyle` (`font` and `fontSize` only when it has a label). An image's `style` leaves out the stroke fields. Colours, widths, roughness and font sizes come back as the names in the style table when one matches and as raw values otherwise. `from` and `to` use a key when the element has one and an element id otherwise. `by` is `agent` for what the CLI created and `user` for what a person drew. Run `show` before touching a canvas you did not just build, and again whenever the user may have changed it. The output is long because every node carries the full style block. Filter it:

```sh
jotura canvas show "Canvases/Launch plan" | jq '.nodes[] | {id, key, text, by}'
jotura canvas show "Canvases/Launch plan" | python -c "import json,sys; [print(n['id'], n.get('key'), n.get('text')) for n in json.load(sys.stdin)['nodes']]"
```

The second line is for a machine without `jq`.

## Looking after the user's drawing

The CLI never moves or changes anything a person drew unless a command targets it. `canvas add` places new nodes in free space, to the right of the element the first new edge connects to, or to the right of everything else when nothing connects. An edge from a new node to a person's element adds the arrow's binding to that element and leaves its position alone. `canvas layout` re-lays out the elements the CLI created, including ones you pinned with `x` and `y`, and leaves the user's elements where they are. `canvas update` and `canvas remove` act on any element you name, and an element a person drew stays theirs afterwards.

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

**Adding to a canvas a person already drew.** Read it first. The user's elements have no key, so connect to them by element id.

```sh
jotura canvas show "Canvases/Launch plan" | jq '.nodes[] | {id, text, by}'
# { "id": "Zk39sLq2", "text": "Launch", "by": "user" }

jotura canvas add "Canvases/Launch plan" --spec - --dry-run --diff <<'EOF'
{
  "nodes": [ { "id": "risk", "text": "Risk: store review", "fill": "yellow" } ],
  "edges": [ { "from": "risk", "to": "Zk39sLq2", "style": "dashed" } ]
}
EOF
```

Read the diff, then run the same command without `--dry-run --diff` and add `--json`. Render the canvas to check it:

```sh
jotura canvas render "Canvases/Launch plan" --out /tmp/plan.png
```

The new box lands to the right of "Launch" and the person's boxes stay put. To retitle your box later: `jotura canvas update "Canvases/Launch plan" --id risk --text "Risk: review delay"`.

## Images

`image` takes the name of a file you pass with `--attach`, or a path or file name of an image already in the vault (resolved like a wikilink):

```sh
jotura canvas add "Canvases/Launch plan" --attach ~/Downloads/dashboard.png --spec - <<'EOF'
{ "nodes": [ { "id": "shot", "image": "dashboard.png", "width": 320 } ] }
EOF
```

On a `.excalidraw.md` canvas the attached file is copied to `assets/<name>` (with a suffix such as `-1` when the name is taken) and the canvas links it there. An image has no border, so put it in a frame or add a box node behind it. A plain `.excalidraw` canvas embeds the bytes. An image over 16 MiB exits 9, and on a plain `.excalidraw` canvas an image over 5 MiB exits 9 too; use a `.excalidraw.md` canvas for large images. A name that is neither attached nor in the vault exits 3. If an image is later missing, `render` draws a placeholder and lists it under `missingImages`.

## Exit codes

| Code | Meaning | What to do |
|---|---|---|
| 2 | `HashConflict`, the canvas changed since your `--if-hash`; the error carries a `currentVsIntended` diff | run `show` for the new hash and state, then re-apply your change |
| 3 | `NotFound`: the canvas, a key, element id or frame name, the spec file, an `--attach` file, a vault image, or an edge endpoint does not exist | run `show` and use a key or id it lists; check the path; for an edge, add the node to the spec or use the key of an element already on the canvas |
| 4 | `Ambiguous`, a bare canvas name matches several canvases | pass the vault path from `candidates` |
| 8 | `PermissionDenied` | tell the user |
| 9 | `InvalidArgs`: a bad spec or flag value, and the message names the field and lists the valid values. Also an edge whose ends overlap, a duplicate spec id, `add` with an id that is already a key, an empty `id`, `text` or `label`, `draw`, `line` or `arrow` points that all sit on one spot, `create` on an existing canvas, `--if-hash` on `create`, a file that is not a canvas, an `update` with nothing to change, an edge asked to move or take a fill, `--stroke`, `--stroke-style` or `--stroke-width` on an image, a `columns` value outside 1 to 20, a `canvas layout --layout` other than `left-right`, `top-down` or `grid` (including `none`), a frame that mixes pinned and laid-out nodes in a spec with several frames, an image over 16 MiB (or over 5 MiB on a plain `.excalidraw` canvas), and a `render --out` that would overwrite an image the canvas uses | fix the field the message names and retry, since nothing was written; for an overlap, move one of the two elements; for a mixed frame, pin all of its nodes or none; for an existing key, pick another id or use `update`; for an image, wrap it in a frame or add a box node behind it; for an existing canvas, use `add`; for `render`, choose another `--out`; for `--layout`, use `left-right`, `top-down` or `grid`; for a large image, shrink it or use a `.excalidraw.md` canvas |
| 14 | `TargetMissing`, the canvas was deleted or moved before the write | check the canvas still exists, then recreate or retry |
| 15 | `ReadOnlyShare`, the canvas, or a `render --out` inside the vault, is in a view-only share | stop and ask the owner for editor access, or render to a path outside the share |
