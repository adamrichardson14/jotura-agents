# Jotura agents

Skills, plugin marketplace and agent instructions for [Jotura](https://jotura.io), the free, local-first markdown notes app your AI agents can work in too.

A Jotura vault is an ordinary folder of `.md` files on your disk. The `jotura` command line gives any file-capable agent (Claude Code, Codex, Gemini CLI, GitHub Copilot, OpenCode) safe, precise operations on that folder: hash-checked edits, strict unique-match replaces, full-text and semantic search, daily notes, and a memory of decisions, facts and follow-ups that survives the session. This repository holds the skills that teach an agent how to use it.

## What is here

| Path | Purpose |
|---|---|
| `jotura/skills/jotura-vault/` | The complete `jotura` CLI reference: reading, searching, the safe edit loop, frontmatter, daily notes, documents, batch edits, exit codes |
| `jotura/skills/jotura-notes/` | Filing and capture rules: where content belongs, how to add to an existing note, the memory loop, what a finished note must contain |
| `jotura/.claude-plugin/plugin.json` | Plugin manifest for Claude Code |
| `.claude-plugin/marketplace.json` | Marketplace manifest so the plugin can be added by repository name |

Both skills are the same files the Jotura desktop app and CLI install, published here so they can be read, reviewed and installed without the app.

## Install

The skills assume the `jotura` binary is on your PATH. It ships inside every Jotura installer and can also be installed on its own:

```sh
curl -fsSL https://jotura.io/install.sh | sh
```

```powershell
irm https://jotura.io/install.ps1 | iex
```

### Claude Code

```
/plugin marketplace add adamrichardson14/jotura-agents
/plugin install jotura@jotura-agents
```

### Codex, Gemini CLI, GitHub Copilot, OpenCode

The CLI writes the skills into each agent's own skill folder:

```sh
jotura skill install codex
jotura skill install gemini
jotura skill install copilot
jotura skill install opencode
```

`jotura skill install claude` does the same for Claude Code without the marketplace, and `--force` rewrites an existing copy.

## How an agent works with a vault

```sh
jotura status --json                       # which vault, is sync on
jotura context "what I am about to do"     # conventions, relevant memories, open follow-ups
HASH=$(jotura read "Projects/Acme.md" --json | jq -r .hash)
jotura edit "Projects/Acme.md" --replace "old line" --with "new line" --if-hash "$HASH"
jotura memory log --kind decision --title "Chose SQLite for the sync db"
```

Every write is atomic, frontmatter is never touched by body edits, and a hash mismatch returns exit code 2 instead of overwriting a change made in the desktop app.

## Learn more

- [Agents on jotura.io](https://jotura.io/agents): what the shared workspace looks like from the human side
- [CLI documentation](https://jotura.io/docs/cli/install): every command and flag
- [Guides](https://jotura.io/guides): persistent memory for Claude Code, Codex, Gemini and Copilot, syncing agent config between machines, and more
- [Download](https://jotura.io/download): the app is free on macOS, Windows, Linux and Android

## Versioning

The plugin version tracks the Jotura CLI release it was published from. Skill files carry a `jotura:begin v=` marker with the same number.

## License

MIT. See [LICENSE](LICENSE).
