# jotura plugin

Two skills that let Claude Code work safely in a [Jotura](https://jotura.io) vault, a folder of plain markdown notes on your disk.

| Skill | What it teaches the agent |
|---|---|
| `jotura-vault` | Every `jotura` CLI command: reading, keyword and semantic search, hash-checked edits that stop the agent overwriting your changes, frontmatter, daily notes, documents and exit codes |
| `jotura-notes` | Where content belongs in the vault, how to add to a note without appending to the bottom, and the memory loop that carries decisions, facts and follow-ups between sessions |

The skills need the `jotura` binary on your PATH. It ships inside the free Jotura app, or on its own:

```sh
curl -fsSL https://jotura.io/install.sh | sh
```

Install the plugin from Claude Code:

```
/plugin marketplace add adamrichardson14/jotura-agents
/plugin install jotura@jotura-agents
```

Full setup, including semantic search and syncing agent config between machines: https://jotura.io/guides/give-your-ai-agents-the-maximum-boost-with-jotura/
