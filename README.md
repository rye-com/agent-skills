# Rye Agent Skills

Agent skills for [Rye's Universal Checkout API](https://docs.rye.com) — turn any product URL into a completed purchase.

Built on the [agentskills.io](https://agentskills.io) open standard. Works with Claude Code, Cursor, OpenClaw, and any compatible agent.

## Skills

| Skill | Description |
|-------|-------------|
| [`rye-overview`](skills/rye-overview/SKILL.md) | Learn what Rye is, explore capabilities, and find the right path for your use case |
| [`rye-universal-checkout`](skills/rye-universal-checkout/SKILL.md) | Integrate Rye checkout into apps or buy products programmatically via AI agents |

## Install

Install with [`skills`](https://www.npmjs.com/package/skills):

```bash
npx skills add rye-com/agent-skills
```

This auto-detects your agent (Claude Code, Cursor, Codex, etc.) and installs the skill to the right location.

### Options

```bash
# Install to a specific agent
npx skills add rye-com/agent-skills -a claude-code

# Install globally (all projects)
npx skills add rye-com/agent-skills -g

# List available skills without installing
npx skills add rye-com/agent-skills --list
```

Once installed, skills activate automatically based on your task — or invoke them manually:

- `/rye-overview` — "What is Rye? What can it do for my use case?"
- `/rye-universal-checkout` — Integrate checkout or buy a product

## Structure

```
skills/
├── rye-overview/
│   └── SKILL.md                  # Capabilities explainer & use case guide
└── rye-universal-checkout/
    ├── SKILL.md                  # Entry point (~110 lines)
    └── references/
        ├── integrate.md          # Developer integration workflow
        ├── buy.md                # Agent commerce workflow
        └── api-reference.md      # Full API reference
```

## License

MIT
