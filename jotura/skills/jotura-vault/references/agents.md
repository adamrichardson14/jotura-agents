# Agent config and skills

Two separate things live on this page, and the desktop app offers them as two
independent opt-ins under Settings, Agents:

- **Sync agent files between devices** (`jotura agents sync`): the vault's
  `Agents/` folder is the canonical copy of instruction files, skills and
  subagent definitions, reconciled against this machine's config directories.
- **Install the Jotura skills** (`jotura skill install`): the two skills Jotura
  ships, their routing blocks, and the Claude session hook.

Either works without the other. With both on, the installed skill folders are
ordinary entries in the registry below, so they travel between devices like
any other skill.

## Agent config in the vault (`jotura agents`)

Global agent config — instruction files, skills, subagent definitions — lives
canonically in the vault under `Agents/`; the system dotfiles are managed
outputs. Edit the vault copy, then run `jotura agents sync` to write it out
(`jotura agents status --json` for a read-only drift report). Never hand-edit
both the vault copy and the system file in one session: a both-sides-changed
file is skipped as a conflict (exit 16, `AgentConflict`) — resolve it with
`jotura agents sync --prefer vault|system`. Nothing on the system side is
deleted without `--prune`.

## Installing the skills in other agents

Two skills ship together: jotura-vault (CLI mechanics) and jotura-notes
(filing and capture principles). `skill install` provisions both, but what
gets written differs by agent:

```sh
jotura skill install claude    # ~/.claude/skills/jotura-vault/{SKILL.md,references/*.md}
                               # + ~/.claude/skills/jotura-notes/SKILL.md
                               # + a routing block in ~/.claude/CLAUDE.md
jotura skill install codex     # ~/.codex/skills/jotura-vault/{SKILL.md,references/*.md}
                               # + ~/.codex/skills/jotura-notes/SKILL.md
                               # + a routing block in ~/.codex/AGENTS.md
jotura skill install opencode  # ~/.config/opencode/skills/jotura-vault/{SKILL.md,references/*.md}
                               # + ~/.config/opencode/skills/jotura-notes/SKILL.md
                               # (no instruction file to amend)
jotura skill install gemini    # a managed block in ~/.gemini/GEMINI.md inlining
                               # notes, skill, and every reference
jotura skill install copilot   # the same inlined block in
                               # ~/.copilot/copilot-instructions.md
jotura skill install all       # all five agents above
jotura skill list              # show install/up-to-date status per artifact
jotura skill show              # print canonical SKILL.md to stdout
jotura skill show --full       # print the combined body: both skills plus
                               # every reference, frontmatter stripped
```

Claude, Codex and OpenCode get the two skill folders on disk and load
references on demand; Gemini and Copilot have no skills directory to follow a
reference into, so their managed block inlines everything (both skill bodies
and all five reference files) into the one instruction file.

Installs amend, never clobber: content outside a managed block
(`<!-- jotura:begin -->` … `<!-- jotura:end -->`) is preserved and re-runs are
idempotent.

**These files are written once and then left alone.** They land when you turn
`Install the Jotura skills` on in the desktop app, or when you run `skill
install` yourself. Nothing rewrites them afterwards: not an app update, not a
newer CLI, not opening a vault, not the install scripts. Taking a newer
version is an explicit act, either the `Reinstall skills` button in Settings,
Agents or `jotura skill install <agent> --force`. Without `--force` an install
leaves a file alone when its block body is already what this binary ships, or
when the stamp on its block is not older than the running binary.

`skill list --json` reports per file: `missing`, `up-to-date`, or `stale`,
where stale means the stamp is older than the running CLI. Nothing learned
about a vault lives in any of these files: that stays in the vault's own Agent
Conventions note (`jotura conventions`), which no install ever touches.
