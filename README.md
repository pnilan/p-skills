# p-skills-airbyte

Personal Claude Code skills, organized as separate plugin marketplaces per branch.

## Marketplaces

Each branch is an independent Claude Code plugin marketplace:

| Branch | Purpose | Install |
|--------|---------|---------|
| `airbyte` | Skills for airbytehq repos | `/plugin marketplace add pnilan/p-skills#airbyte` |
| `personal` | Skills for personal projects | `/plugin marketplace add pnilan/p-skills#personal` |

## Adding a new marketplace

1. Create a new branch from `main`
2. Add `.claude-plugin/marketplace.json` at the root
3. Create a plugin directory with `.claude-plugin/plugin.json` and `skills/`
4. Push the branch
