# Images and other binary files

Notes embed images with `![[diagram.png]]` or `![](Assets/diagram.png)`. The
image bytes live in the vault as an ordinary file; the note only holds the
link. You can look at an image yourself: get its absolute path from the CLI,
then open that path with your harness's own image reader (Claude Code's
`Read` tool renders an image file when given its path; other harnesses have
an equivalent). The CLI never prints image bytes.

## Find the images a note embeds

```sh
jotura links projects/x.md --json
# → { "source": "projects/x.md", "links": [
#      { "line": 12, "linkKind": "wiki", "target": "diagram.png",
#        "resolved": ["Attachments/diagram.png"],
#        "absolutePath": "/Users/me/vault/Attachments/diagram.png", ... } ] }
```

Every link the note contains is listed, notes and files alike. `resolved` is
the vault-relative path the target resolves to (best match first; an embed
resolves by filename, then the Obsidian attachment folder, then the note's
own folder). `absolutePath` is present only when the link resolved; a link
with an empty `resolved` list is dead, so do not guess a path for it.

## Get the absolute path of one file

```sh
jotura read Attachments/diagram.png --json
# → { "path": "Attachments/diagram.png",
#     "absolutePath": "/Users/me/vault/Attachments/diagram.png",
#     "kind": "image", "mime": "image/png", "bytes": 48213 }

jotura read Attachments/diagram.png
# → Binary file (image/png, 48 KB): /Users/me/vault/Attachments/diagram.png. Open it with an image-capable tool.
```

`read` describes a binary file instead of printing it: `kind` is `image`
for png, jpg, jpeg, gif, webp, svg, bmp, avif, ico, heic, heif and tiff, and
`binary` for everything else (audio, video, PDF and office documents). A
PDF or office document has a searchable markdown mirror; read that instead
(`references/documents.md`). No `hash` or `body` is returned, and `edit`
refuses binary paths.

## Look at the image

Pass `absolutePath` to your image-capable file tool. In Claude Code that is
the `Read` tool with the absolute path; the image is shown to you like a
screenshot. Do not `cat` the file and do not base64 it into the
conversation.

## Add an image to a note

1. Copy the file into the vault's image folder with your shell. The
   desktop app puts pasted images in `assets/` (documents imported with
   `jotura import` go to `attachments/`), unless the vault is an Obsidian
   vault with `attachmentFolderPath` set in `.obsidian/app.json`, in which
   case use that folder (`./` prefixed values mean a subfolder beside the
   note). Follow whichever folder the vault already uses, and ask the user
   before creating a new one.
2. Insert the embed into the note with the normal edit loop, at the place
   it belongs:

```sh
cp ~/Downloads/diagram.png "$VAULT/assets/diagram.png"
HASH=$(jotura read projects/x.md --json | jq -r .hash)
jotura edit projects/x.md --insert-after 12 --content "![[diagram.png]]" --if-hash "$HASH"
```

A bare filename in `![[...]]` is enough when it is unique in the vault;
write the folder (`![[assets/diagram.png]]`) when two files share a
name.

## Sizing

`![[diagram.png|300]]` renders the image 300 pixels wide; `![[diagram.png|300x200]]`
sets width and height. The size is part of the link text and is kept in the
file. `jotura links` reports the target without the size suffix.
