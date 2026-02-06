# Add Candidates Skill

Add one or more hiring candidates to a Notion database from any source — Gmail emails, Slack messages, direct input, or other sources.

## Usage

```
/add-candidates [source-hint]
```

Examples:
```
/add-candidates
/add-candidates from Gmail
/add-candidates John Smith backend engineer
```

## Features

- Extracts candidate info from multiple sources (Gmail, Slack, paste, URLs, files)
- Handles single or batch candidate entries
- Dynamically discovers any Notion database schema — not hardcoded to a specific DB
- Maps extracted fields to the target database properties with user confirmation
- Optionally drafts a reply to the sender (Gmail reply, Slack thread, or plain text)

## Prerequisites

**Required:** Notion MCP server configured in `~/.claude.json`:

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer ntn_xxx\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

**Optional (based on source):**

- **Chrome DevTools MCP** — needed for Gmail/web-based extraction
- **Slack MCP** — needed for Slack message extraction

The skill checks prerequisites at startup and only requires MCPs relevant to your chosen source.

## Installation

```bash
cp -r productivity/add-candidates ~/.claude/skills/
```
