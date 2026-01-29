# Daily Prep Skill

Prepare for daily standup meetings by analyzing Linear issues and identifying deviations from commitments.

## Usage

```
/daily-prep [project-name]
```

Example:
```
/daily-prep rain-clearhaven
```

## Features

- Analyzes yesterday's commitments (issues due yesterday) and checks completion status
- Lists today's commitments (issues due today)
- Flags issues in Todo/In Progress without due dates (missing commitments)
- Reviews scheduled meetings and checks for agenda, owner, and prep tasks
- Identifies blockers and dependencies
- Generates a copy-paste ready prep sheet

## Prerequisites

Requires Linear MCP server to be configured in `~/.claude.json`:

```json
{
  "mcpServers": {
    "linear": {
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "your-api-key-here"
      }
    }
  }
}
```

Get your Linear API key from: https://linear.app/settings/api

## Installation

```bash
cp -r productivity/daily-prep ~/.claude/skills/
```
