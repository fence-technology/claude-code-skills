---
name: daily-prep
description: Prepare for project daily meetings as a project lead. Reviews yesterday's commitments, compares against current status, identifies deviations and questions to ask the team.
argument-hint: [project-name]
disable-model-invocation: true
allowed-prompts:
  - tool: Bash
    prompt: "parse Linear API JSON responses with jq or python"
---

# Daily Meeting Prep

Help the project lead prepare for daily standup meetings by analyzing Linear issues and identifying deviations from commitments.

## Phase 0: Pre-requisite Check

First, verify Linear MCP is available by attempting to list teams:

1. Call `mcp__linear__list_teams`
2. If the call fails or Linear MCP is not configured:
   - Display this error message:
   ```
   ## Linear MCP Not Configured

   This skill requires the Linear MCP server to be configured.

   ### Setup Instructions:
   1. Get your Linear API key from: https://linear.app/settings/api
   2. Add the Linear MCP server to your Claude settings (~/.claude.json):
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
   3. Restart Claude Code
   4. Run /daily-prep again
   ```
   - Stop execution here
3. If successful, proceed to Phase 1

## Phase 1: Context Gathering

1. If a project name was provided as argument, use it to filter projects
2. Call `mcp__linear__list_projects` to list available projects
3. If no argument provided or multiple matches, ask the user to select from the list using AskUserQuestion
4. Once project is selected, call `mcp__linear__list_issues` with the project filter to get all issues
5. Call `mcp__linear__list_cycles` to get current sprint/cycle context

### Handling Linear API Response Issues

The Linear API may return errors or large responses that need special handling:

- **"Too many subrequests" error**: Reduce the `limit` parameter (try 100 instead of 250)
- **Response saved to file**: When output exceeds token limits, the MCP server saves results to a file. Use `jq` or `python` to parse the JSON and extract relevant fields:
  ```bash
  cat "<saved-file-path>" | python3 -c "
  import json, sys
  data = json.load(sys.stdin)
  issues = json.loads(data[0]['text'])['issues']
  for i in issues:
      print(f\"{i['identifier']} | {i.get('dueDate')} | {i.get('status')} | {i.get('assignee', 'Unassigned')}\")
  "
  ```

### Data Quality Checks

When processing issues, also flag potential gaps:
- **Issues in Todo/In Progress without a due date**: These represent uncommitted work that may slip. Include them in a "Missing Commitments" section in the output so the team can assign due dates.

## Phase 2: Yesterday's Commitment Analysis

Calculate yesterday's date and analyze commitments:

1. From the fetched issues, identify those with **due date = yesterday**
   - These represent yesterday's commitments
2. Check each issue's current status:
   - If status is "Done" or "Completed" -> commitment met
   - If status is anything else (In Progress, To Do, etc.) -> deviation, needs follow-up
3. For each deviation, note:
   - Issue ID and title
   - Assignee
   - Current status
   - Generate a question to ask the assignee

## Phase 3: Today's Interactions Review

Use AskUserQuestion to gather information about today's planned meetings:

1. Ask: "Do you have any meetings scheduled today (client or internal)? List them if yes."

2. **For each meeting mentioned**, follow up with deeper questions:
   - "Is there a Linear issue or task created to prepare for [meeting name]?"
   - "Does [meeting name] have a clear agenda defined?"
   - "Who is the owner/driver of [meeting name]?"

3. Search Linear issues for any that might be related to mentioned meetings (e.g., issues with "meeting", "sync", "call" in title or related to mentioned clients)

4. From the project issues, identify any that are:
   - Marked as blocked
   - Have dependencies on other teams/issues
   - Have comments indicating coordination needed

## Phase 4: Generate Prep Output

Generate a structured markdown prep sheet with the following sections:

```markdown
## Daily Prep: [Project Name] - [Today's Date]

### Yesterday's Commitments (due [yesterday's date])
| Issue | Title | Assignee | Status | Completed? |
|-------|-------|----------|--------|------------|
| [ID] | [Title] | @[assignee] | [status] | YES/NO - ASK |

### Questions to Ask
- @[assignee]: [Issue ID] "[title]" was due yesterday but is still [status] - what blocked completion?
- [Additional questions based on deviations]

### Today's Commitments (due [today's date])
| Issue | Title | Assignee | Status |
|-------|-------|----------|--------|
| [ID] | [Title] | @[assignee] | [status] |

### Missing Commitments (no due date)
| Issue | Title | Assignee | Status |
|-------|-------|----------|--------|
| [ID] | [Title] | @[assignee] | [status] |

> These issues are in Todo/In Progress but have no due date. Consider assigning dates to track commitments.

### Today's Meetings
For each meeting, verify:
- [ ] [Meeting name] - Agenda: [YES/NO] | Owner: [name] | Prep task: [issue ID or NONE]

### Blockers & Dependencies
- [Issue ID] blocked by [reason/blocker description]
- [Issue ID] waiting on [dependency]

### Definition Items (topics needing discussion)
- [Any issues with unclear requirements or missing information]
- [Decisions that need to be made]
```

## MCP Tools Used

- `mcp__linear__list_teams` - Verify Linear MCP is configured
- `mcp__linear__list_projects` - List available projects for selection
- `mcp__linear__list_issues` - Fetch issues by project with filters
- `mcp__linear__get_issue` - Get detailed issue info if needed
- `mcp__linear__list_cycles` - Get current sprint/cycle context

## Glossary

- **FOD** = Fence Onboarding Document (not Dashboard)

## Important Notes

- Always use ToolSearch to load Linear MCP tools before calling them
- Format dates appropriately for the user's locale
- Keep questions concise and actionable
- Focus on deviations - don't overwhelm with completed items
- The prep sheet should be copy-paste ready for notes
