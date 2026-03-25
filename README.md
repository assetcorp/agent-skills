# Agent Skills

A collection of reusable skills that extend what LLM agents can do. Each skill teaches an agent how to handle a specific domain through structured markdown instructions, making them portable across any agent framework that supports skill files.

## Skills

| Skill | Description | Frameworks / References |
| ----- | ----------- | ----------------------- |
| [cli-builder](skills/cli-builder/) | Build CLIs that work for humans, agents, and automation. Includes a 12-point machine-friendly checklist covering non-interactive design, structured output, actionable errors, idempotency, and piping support. | Click/Typer (Python), Commander (Node.js/TS), Cobra (Go), Clap (Rust) |

## Installation

Install any skill from this repo using [`npx skills`](https://github.com/vercel-labs/skills):

```bash
# Install a specific skill
npx skills add assetcorp/agent-skills --skill cli-builder

# Install all skills from this repo
npx skills add assetcorp/agent-skills --all

# Install to a specific agent
npx skills add assetcorp/agent-skills --skill cli-builder -a claude-code -a codex -a cursor
```

The CLI detects skills, copies the files locally, and makes them available to your agent. It works with Claude Code, Codex, Cursor, GitHub Copilot, OpenCode, and other compatible agents.

## License

[MIT](LICENSE)
