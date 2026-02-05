# Claude Code Skills

Shared Claude Code skills for the Fence team.

## Structure

Skills are organized by category:

```
/productivity     - Daily workflows, meeting prep, time management
/engineering      - Code review, debugging, deployment helpers, log investigation
```

## Installation

To install a skill, copy the skill folder to your local Claude skills directory:

```bash
# Example: Install the project-daily-prep skill
cp -r productivity/project-daily-prep ~/.claude/skills/
```

Or clone the entire repo and symlink:

```bash
git clone https://github.com/fence-technology/claude-code-skills.git ~/claude-code-skills
ln -s ~/claude-code-skills/productivity/project-daily-prep ~/.claude/skills/project-daily-prep
```

## Available Skills

### Productivity

| Skill | Description | Usage |
|-------|-------------|-------|
| [project-daily-prep](productivity/project-daily-prep) | Prepare for daily standups by analyzing Linear issues and identifying deviations | `/project-daily-prep [project-name]` |

### Engineering

| Skill | Description | Usage |
|-------|-------------|-------|
| [cloudwatch-investigate](engineering/cloudwatch-investigate) | Investigate CloudWatch logs. Specify the error type (e.g., asset declaration, intake creation, metric calculation, webhook processing) and optionally a deal_id or other identifier. Auto-routes to the correct log group, traces errors to source code, and suggests fixes. | `/cloudwatch-investigate [deal_id] <error_type>` |

## Contributing

1. Create a new folder under the appropriate category
2. Add a `SKILL.md` file following the [Claude Code skill format](https://docs.anthropic.com/en/docs/claude-code)
3. Submit a PR with a description of the skill

## Requirements

Some skills require MCP servers to be configured. Check each skill's documentation for prerequisites.
