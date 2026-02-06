---
name: add-candidates
description: Add one or more hiring candidates to a Notion database. Reads candidate info from any source (email, message, or direct input), maps it to the target Notion DB schema, creates entries, and optionally drafts a reply. Use when adding candidates to hiring pipelines.
argument-hint: [source-hint]
disable-model-invocation: true
---

# Add Candidates to Notion Hiring Database

You are executing the `add-candidates` skill. Follow these phases sequentially.

---

## Pre-requisites Check

Before doing anything else, verify that the required MCP servers are available and responding. Run these checks and report the results to the user:

1. **Notion MCP** (always required): Use `ToolSearch` to find and load `mcp__notion__notion-search`. Then call `mcp__notion__notion-search` with a simple query (e.g., `query: "test"`) to confirm it responds. If this fails, **stop immediately** and tell the user: *"Notion MCP server is not available. Please make sure it's configured and running before using this skill."*

If Notion MCP is ready, proceed to Phase 0. Source-specific MCP checks (Chrome DevTools, Slack, etc.) happen later in Phase 1, only after the user has chosen their input source.

---

## Phase 0: Determine Input Source

Check if `$ARGUMENTS` was provided. If so, use it as a hint for locating the candidate info (e.g., an email subject line, a candidate name, "from Gmail", etc.).

Ask the user how they want to provide the candidate information. Present these options:

1. **Gmail email** - You'll navigate Chrome to Gmail, search for the email, and extract candidate details
2. **Slack message** - You'll use Slack MCP tools to find and read the message
3. **Paste / dictate** - The user provides candidate info directly in chat
4. **Other** - The user points to another source (document, URL, file, etc.)

If `$ARGUMENTS` makes the source obvious (e.g., contains "Gmail" or an email subject), suggest that option but still confirm with the user.

---

## Phase 1: Extract Candidate Information

Before extracting, **check that the required MCP for the chosen source is available**:
- **Gmail / web source**: Use `ToolSearch` to find `mcp__chrome-devtools__navigate_page`. If not available, tell the user Chrome DevTools MCP is needed and ask them to either configure it or choose a different source.
- **Slack source**: Use `ToolSearch` to find tools matching `slack`. If not available, tell the user Slack MCP is needed and ask them to either configure it or choose a different source.
- **Paste/dictate or file sources**: No additional MCP needed.

Once the source-specific MCP is confirmed, proceed with extraction:

### Gmail email
1. Use Chrome DevTools MCP tools to navigate to `https://mail.google.com`
2. Wait for Gmail to load, then search for the relevant email (use `$ARGUMENTS` as the search query if provided)
3. Open the email and take a snapshot of the content
4. Extract all candidate details: name, email, profile/CV link, source, notes, and any other relevant info

### Slack message
1. Use `mcp__slack__search_messages` or similar Slack MCP tools to find the message
2. Read the full message content and any thread replies
3. Extract candidate details from the message

### Paste / dictate
1. Ask the user to provide the candidate information
2. Parse the provided text for candidate details

### Other sources
1. Adapt based on what the user describes:
   - For URLs: use `WebFetch` to retrieve content
   - For web apps: use Chrome DevTools MCP tools to navigate and extract
   - For files: use `Read` to read the file content
2. Extract candidate details from the content

### After extraction (all sources)
- **Handle multiple candidates**: If the source contains info about more than one candidate, extract ALL of them
- Present the extracted candidate info to the user in a clear format for confirmation
- Ask the user to correct anything that's wrong or add missing info
- Confirm before proceeding

---

## Phase 2: Determine Target Notion Database

1. Ask the user which Notion database to add the candidate(s) to. If `$ARGUMENTS` or conversation context hints at a specific DB, suggest it.

2. Use `mcp__notion__notion-search` to find the database by name:
   ```
   query: "<database name>"
   filter: { "property": "object", "value": "database" }
   ```

3. Use `mcp__notion__notion-fetch` to retrieve the database schema:
   - Fetch the database URL/ID to get its properties
   - Identify all property names, types, and available options (especially for select/multi-select/status fields)

4. Map the extracted candidate fields to the database properties:
   - Auto-map fields where the value can be inferred from the source (e.g., "Name" -> title property, "Email" -> email property, "CV Link" -> URL property)
   - For select/status/multi-select fields, show the available options and ask the user to pick the right values
   - **People/assignee fields (e.g., "Process Lead", "Hiring Manager", "Recruiter") can never be inferred from candidate info — always ask the user explicitly.** Use `mcp__notion__notion-get-users` to list available workspace members and let the user pick.
   - For any other fields that don't have an obvious mapping, ask the user

5. Present the full mapping to the user for confirmation:
   ```
   Candidate: [Name]
   -> [DB Property 1]: [value]
   -> [DB Property 2]: [value]
   ...
   ```

6. Wait for user confirmation before proceeding.

---

## Phase 3: Create Notion Entries

1. Use `mcp__notion__notion-create-pages` to create the candidate page(s):
   - Build the properties object according to the confirmed mapping
   - For multiple candidates, create all entries (batch if the API supports it, otherwise create sequentially)

2. After creation, use `mcp__notion__notion-fetch` to verify the created page(s):
   - Confirm all fields are correctly populated
   - Retrieve the page URL(s)

3. Show the user:
   - Confirmation that the entry/entries were created
   - The Notion page URL(s) for each candidate

---

## Phase 4: Optional Reply

Ask the user: **"Do you want to draft a reply to the sender?"**

If **no**, the skill is complete. Summarize what was done and end.

If **yes**:

1. Ask the user what key points to include in the reply (e.g., "confirming receipt", "will review this week", custom message)

2. Draft the reply based on the source:

### Gmail reply
- Use Chrome DevTools MCP to navigate back to the email (if not already there)
- Click "Reply All" (or "Reply" based on user preference)
- Use the `fill` tool to type the drafted reply into the compose area
- Show the user the draft and ask for confirmation before sending (do NOT click Send automatically)

### Slack reply
- Use Slack MCP tools to post a reply in the same thread
- Show the draft to the user first and confirm before sending

### Other sources
- Compose the reply text and display it to the user
- Let the user copy and send it manually

---

## Important Notes

- **Never hardcode database names or field mappings** - always discover them dynamically via the Notion API
- **Always confirm with the user** before creating entries or sending replies
- **Handle errors gracefully** - if a Notion API call fails, explain what went wrong and suggest fixes
- **Respect rate limits** - if creating many candidates, add brief pauses between API calls
