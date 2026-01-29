# Claude Code Skills

Shared Claude Code skills for the Fence team.

## Structure

Skills are organized by category:

```
/productivity     - Daily workflows, meeting prep, time management
/engineering      - Code review, debugging, deployment helpers
/integrations     - Third-party service integrations (Linear, Notion, etc.)
```

## Installation

To install a skill, copy the skill folder to your local Claude skills directory:

```bash
# Example: Install the daily-prep skill
cp -r productivity/daily-prep ~/.claude/skills/
```

Or clone the entire repo and symlink:

```bash
git clone https://github.com/fence-technology/claude-code-skills.git ~/claude-code-skills
ln -s ~/claude-code-skills/productivity/daily-prep ~/.claude/skills/daily-prep
```

## Available Skills

### Productivity

| Skill | Description | Usage |
|-------|-------------|-------|
| [daily-prep](productivity/daily-prep) | Prepare for daily standups by analyzing Linear issues and identifying deviations | `/daily-prep [project-name]` |

## Contributing

1. Create a new folder under the appropriate category
2. Add a `SKILL.md` file following the [Claude Code skill format](https://docs.anthropic.com/en/docs/claude-code)
3. Submit a PR with a description of the skill

## Requirements

Some skills require MCP servers to be configured. Check each skill's documentation for prerequisites.
