---
name: cloudwatch-investigate
description: Investigate CloudWatch logs for a deal. Automatically detects the appropriate log group based on the query context (API, assets processor, lambdas, etc.). Can trace errors back to source code and suggest fixes.
argument-hint: [deal_id] <description of what to investigate>
disable-model-invocation: true
allowed-prompts:
  - tool: Bash
    prompt: "run docker with AWS CLI to query CloudWatch logs"
  - tool: Bash
    prompt: "parse CloudWatch JSON responses with python"
  - tool: Bash
    prompt: "start Docker Desktop"
---

# CloudWatch Log Investigation

Investigate CloudWatch logs for a specific deal, automatically routing to the appropriate log group based on the investigation context. When errors are found, trace them back to the source code and suggest fixes.

## Phase 0: Pre-requisite Check

### Docker

1. Check if Docker is running:
   ```bash
   docker info &>/dev/null && echo "Docker is running" || echo "Docker not running"
   ```

2. If Docker is not running, start it:
   ```bash
   open -a Docker
   # Wait for Docker to be ready
   for i in {1..30}; do docker info &>/dev/null && echo "Docker ready" && break || sleep 2; done
   ```

### AWS Credentials via Leapp

This skill requires AWS credentials. We use **Leapp** to manage AWS credentials securely.

1. Check if credentials exist:
   ```bash
   cat ~/.aws/credentials 2>/dev/null | head -5
   ```

2. If no credentials or expired, display:
   ```
   ## AWS Credentials Not Found or Expired

   This skill requires AWS credentials configured via Leapp.

   ### Setup Instructions:

   1. **Open Leapp** application (download from https://www.leapp.cloud if not installed)

   2. **Start a session** for the Fence AWS account:
      - Find the "fence-prod" or appropriate profile
      - Click the play button to start the session
      - This will populate ~/.aws/credentials with temporary credentials

   3. **Verify credentials** are active:
      ```bash
      cat ~/.aws/credentials | head -5
      ```
      You should see `aws_access_key_id`, `aws_secret_access_key`, and `aws_session_token`

   4. **Run /cloudwatch-investigate again**

   ### Troubleshooting:
   - If credentials show but are expired, stop and restart the Leapp session
   - Ensure you're using the correct AWS account (prod vs dev)
   - Session tokens expire after 1 hour by default
   ```

## Phase 1: Parse Arguments

1. Check if the first argument is a UUID (format: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
   - If yes, use it as the `deal_id` and extract the investigation description from remaining arguments
   - If no, treat the entire argument as the investigation description (no deal_id filter)
2. The `deal_id` is **optional** - searches can be performed without it using other identifiers:
   - `asset_external_id` (e.g., "33112")
   - `contract_external_id` (e.g., "5341")
   - `correlation_id`
   - Error messages or keywords
3. If no deal_id and no other identifier is clear from the prompt, ask the user:
   - "What identifier should I search for? (deal_id, asset_external_id, contract_external_id, or keyword)"

## Phase 2: Determine Log Group

Based on keywords in the investigation description, select the appropriate log group(s):

### Log Group Mapping

| Keywords | Log Group | Description |
|----------|-----------|-------------|
| api, request, endpoint, asset declaration, contract, client, PUT, POST, GET, 500, 400 | `/aws/fargate/api/prod` | Main API requests |
| assets processor, asset processing, intake | `/aws/fargate/assets_processor/prod` | Asset processing service |
| internal api, internal | `/aws/fargate/backend-internal-api/prod` | Internal API |
| executor | `/aws/fargate/executor/prod` | Executor service |
| long running, action, step function, workflow | `/aws/fargate/long-running-service/prod` | Long-running workflows |
| metric, calculation, borrow base | `/aws/lambda/fence-backend-prod-metric-calculator` | Metric calculations |
| reconciliation | `/aws/lambda/fence-backend-prod-reconciliation` | Reconciliation jobs |
| covenant | `/aws/lambda/fence-backend-prod-covenant-checker` | Covenant checks |
| extraction | `/aws/lambda/fence-backend-prod-extraction-processor` | Document extraction |
| webhook, bridge | `/aws/lambda/fence-backend-prod-bridge_webhook_processor` | Bridge webhooks |
| webhook, monerium | `/aws/lambda/fence-backend-prod-monerium_webhook_processor` | Monerium webhooks |
| deposit, withdrawal, repayment | `/aws/vendedlogs/states/fence-backend-prod-*-logs` | Step Function logs |

### Default Behavior

- If no specific keywords match, default to `/aws/fargate/api/prod` (most common)
- If the user mentions "error" or "failed", also search for ERROR level logs specifically

## Phase 3: Execute CloudWatch Query

### Basic Query Template

```bash
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli logs filter-log-events \
  --log-group-name "<LOG_GROUP>" \
  --filter-pattern "<DEAL_ID>" \
  --start-time $(date -v-24H +%s)000 \
  --output json
```

### Time Range Selection

Ask the user or infer from context:
- "last hour" → `-v-1H`
- "last 6 hours" → `-v-6H`
- "last 24 hours" → `-v-24H` (default)
- "last 7 days" → `-v-7d`
- "today" → use midnight as start time

### Filtering by Log Level

For error investigation, add filter pattern:
```bash
--filter-pattern '"<DEAL_ID>" "ERROR"'
```

## Phase 4: Parse and Format Results

Use Python to parse and format the CloudWatch JSON response:

```bash
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli logs filter-log-events \
  --log-group-name "<LOG_GROUP>" \
  --filter-pattern "<DEAL_ID>" \
  --start-time $(date -v-24H +%s)000 \
  --output json 2>&1 | python3 << 'EOF'
import json
import sys
from datetime import datetime

data = json.load(sys.stdin)
for event in sorted(data.get('events', []), key=lambda x: x['timestamp']):
    msg = json.loads(event['message'])
    ts = event['timestamp']
    dt = datetime.utcfromtimestamp(ts/1000).strftime('%Y-%m-%d %H:%M:%S UTC')
    level = msg.get('levelname', 'INFO')
    message = msg.get('message', '')

    print(f'[{dt}] [{level}] {message}')

    # Print request details if available
    if 'metadata' in msg and msg['metadata']:
        meta = msg['metadata']
        if 'request_body' in meta:
            print(f'  Request: {meta.get("method", "")} {meta.get("url_path", "")}')
            print(f'  Status: {meta.get("status_code", "")}')
            if meta.get('query_params'):
                print(f'  Query: {meta.get("query_params")}')
            if isinstance(meta.get('request_body'), dict):
                for k, v in list(meta['request_body'].items())[:10]:
                    print(f'    {k}: {v}')

    # Print stack trace if available
    if 'exc_info' in msg:
        print(f'  Stack trace:')
        for line in msg['exc_info'].split('\n'):
            print(f'    {line}')
    print()
EOF
```

## Phase 5: Trace Errors to Source Code

**CRITICAL: When errors are found in logs, you MUST trace them back to the actual source code in the fence-backend repository.**

### Stack Trace Analysis

When you find an error with a stack trace like:
```
File "/code/.venv/lib/python3.10/site-packages/fence/core_lib/deals/mta/services/insights.py", line 923, in _build_absolute_cap_sector_constraint
    if sector_range["min_cnae"] <= asset.sector_code <= sector_range["max_cnae"]:
TypeError: '<=' not supported between instances of 'str' and 'NoneType'
```

You MUST:

1. **Map the deployed path to the local source path**:
   - `/code/.venv/lib/python3.10/site-packages/fence/core_lib/` → `source/fence-core-lib/fence/core_lib/`
   - `/code/app/` → `source/app/`

2. **Read the actual source file** using the Read tool:
   ```
   Read: source/fence-core-lib/fence/core_lib/deals/mta/services/insights.py
   ```

3. **Navigate to the specific line number** mentioned in the stack trace

4. **Understand the context** by reading surrounding code (±50 lines)

5. **Identify the root cause** by analyzing:
   - What data type was expected vs what was received
   - What validation is missing
   - What edge case wasn't handled

### Path Mapping Reference

| Deployed Path | Local Source Path |
|---------------|-------------------|
| `/code/.venv/.../fence/core_lib/` | `source/fence-core-lib/fence/core_lib/` |
| `/code/app/` | `source/app/` |
| `/code/app/api/` | `source/app/api/` |

### Common Error Patterns

| Error Type | Likely Cause | Where to Look |
|------------|--------------|---------------|
| `TypeError: ... 'NoneType'` | Missing data, null field | Check data validation, model fields |
| `KeyError` | Missing dict key | Check API request schema, data transformations |
| `IntegrityError` | DB constraint violation | Check unique constraints, foreign keys |
| `ValidationError` | Pydantic/schema validation | Check request models in `app/api/*/schemas/` |
| `AttributeError` | Object missing attribute | Check model definitions, None checks |

## Phase 6: Generate Investigation Report with Fix Suggestion

Present findings in a structured format:

```markdown
## CloudWatch Investigation: [Deal ID]

**Log Group:** [selected log group]
**Time Range:** [start] to [end]
**Total Events:** [count]

### Summary

[Brief summary of what was found]

### Timeline

| Time (UTC) | Level | Event | Status |
|------------|-------|-------|--------|
| [timestamp] | [INFO/ERROR/WARNING] | [description] | [status code if applicable] |

### Errors Found

[If any errors, list them with full stack traces]

**Error:** [error type and message]
**File:** [local source path]:[line number]

### Root Cause Analysis

**What happened:**
[Explain the error in plain terms]

**Why it happened:**
[Explain the underlying cause - missing data, edge case, etc.]

**Affected code:**
```python
# source/[path/to/file.py]:[line_number]
[show the problematic code snippet]
```

### Suggested Fix

**Option 1: [Fix name]**
```python
# In source/[path/to/file.py], around line [X]
# Change:
[old code]

# To:
[new code with fix]
```

**Explanation:** [Why this fix addresses the root cause]

**Option 2: [Alternative fix if applicable]**
[...]

### Request Details

[For API investigations, show request/response details]

### Recommended Actions

1. [ ] [Specific action with file path]
2. [ ] [Test command to verify fix]
3. [ ] [Any data fixes needed]
```

## Advanced Queries

### Search by Correlation ID

If you find an interesting request and need to trace all related logs:

```bash
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli logs filter-log-events \
  --log-group-name "<LOG_GROUP>" \
  --filter-pattern '"<CORRELATION_ID>"' \
  --start-time $(date -v-24H +%s)000 \
  --output json
```

### Search by External ID

For asset or contract investigations:

```bash
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli logs filter-log-events \
  --log-group-name "/aws/fargate/api/prod" \
  --filter-pattern '"asset_external_id" "<EXTERNAL_ID>"' \
  --start-time $(date -v-24H +%s)000 \
  --output json
```

### Search Multiple Log Groups

For complex investigations spanning multiple services, run queries in parallel and combine results.

## Available Log Groups Reference

### ECS/Fargate Services
- `/aws/fargate/api/prod` - Main API
- `/aws/fargate/assets_processor/prod` - Asset processing
- `/aws/fargate/backend-internal-api/prod` - Internal API
- `/aws/fargate/executor/prod` - Executor
- `/aws/fargate/long-running-service/prod` - Long-running workflows

### Key Lambda Functions
- `/aws/lambda/fence-backend-prod-assets-processor`
- `/aws/lambda/fence-backend-prod-metric-calculator`
- `/aws/lambda/fence-backend-prod-metric-gatherer`
- `/aws/lambda/fence-backend-prod-reconciliation`
- `/aws/lambda/fence-backend-prod-borrow-base`
- `/aws/lambda/fence-backend-prod-covenant-checker`
- `/aws/lambda/fence-backend-prod-extraction-processor`
- `/aws/lambda/fence-backend-prod-*_webhook_processor` (bridge, bvnk, monerium, stripe, slack)

### Step Functions
- `/aws/vendedlogs/states/fence-backend-prod-deposit-logs`
- `/aws/vendedlogs/states/fence-backend-prod-withdrawal-logs`
- `/aws/vendedlogs/states/fence-backend-prod-repayment-logs`
- `/aws/vendedlogs/states/fence-backend-prod-cashflow-logs`

## Important Notes

- Always use Docker with mounted AWS credentials: `-v ~/.aws:/root/.aws`
- Default region is `eu-central-1` (set in credentials)
- Timestamps in logs are in UTC
- Large result sets may need pagination - use `--limit` and `--next-token`
- For very recent events (< 1 minute), there may be ingestion delay
- **Always trace errors to source code** - don't just report the error, explain it
- **Suggest fixes** when the user asks for solutions or when the fix is obvious
