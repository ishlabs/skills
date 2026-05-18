# ishlabs/skills

Agent Skills for the [ish](https://ishlabs.io) CLI, in the [Agent Skills](https://agentskills.io/) format.

## Install

### `npx skills` (any supported agent)

The [skills CLI](https://github.com/vercel-labs/skills) auto-detects installed agents and falls back to a prompt when none are detected.

```bash
# Install every skill in this repo
npx skills add ishlabs/skills

# Install one specific skill
npx skills add ishlabs/skills --skill ish

# List what's available without installing
npx skills add ishlabs/skills --list
```

### Per-agent commands

| Agent | Install command |
|-------|-----------------|
| Claude Code (plugin marketplace) | `/plugin marketplace add ishlabs/skills` then `/plugin install ish@ishlabs` |
| Claude Code (skills CLI) | `npx skills add ishlabs/skills -a claude-code` |
| Codex | `npx skills add ishlabs/skills -a codex` |
| Cursor | `npx skills add ishlabs/skills -a cursor` |
| OpenCode | `npx skills add ishlabs/skills -a opencode` |
| Cline | `npx skills add ishlabs/skills -a cline` |
| Roo Code | `npx skills add ishlabs/skills -a roo` |
| Gemini CLI | `npx skills add ishlabs/skills -a gemini-cli` |
| Amp | `npx skills add ishlabs/skills -a amp` |
| Windsurf | `npx skills add ishlabs/skills -a windsurf` |

For other agents, run `npx skills add ishlabs/skills` and pick from the prompt, or pass `-a <agent>` for any entry in the [skills CLI matrix](https://github.com/vercel-labs/skills#supported-agents).

## Skills

- [`ish`](skills/ish/SKILL.md) — Drive the ish CLI for studies, asks, simulations, and chatbot probes.

Built for [ish](https://ishlabs.io). CLI source: <https://github.com/ishlabs/ish-cli>.

## Source of truth

The contents of [`skills/ish/`](skills/ish/) are generated from [`src/lib/skill-content.ts`](https://github.com/ishlabs/ish-cli/blob/main/src/lib/skill-content.ts) in the `ish-cli` repo and regenerated on every CLI release. Edits made directly here will be overwritten.

Bug reports about the `ish` skill belong at <https://github.com/ishlabs/ish-cli/issues>. Bug reports about this repo's plumbing (install matrix, marketplace manifest, CI) belong here.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE).
