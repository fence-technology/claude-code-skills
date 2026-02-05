# CloudWatch Investigate Skill

Quickly investigate CloudWatch logs for a specific deal, automatically routing to the appropriate log group based on your query context. When errors are found, traces them back to the source code and suggests fixes.

## Usage

```
/cloudwatch-investigate <deal_id> [what to investigate]
```

### Examples

```bash
# Investigate API errors for a deal
/cloudwatch-investigate 57af6a49-74d2-41cf-bd5b-1e98c03df11d api errors

# Check asset declarations
/cloudwatch-investigate 57af6a49-74d2-41cf-bd5b-1e98c03df11d asset declaration failures

# Investigate metric calculations
/cloudwatch-investigate 57af6a49-74d2-41cf-bd5b-1e98c03df11d metric calculation

# Check webhook processing
/cloudwatch-investigate 57af6a49-74d2-41cf-bd5b-1e98c03df11d monerium webhook

# General investigation (defaults to API logs)
/cloudwatch-investigate 57af6a49-74d2-41cf-bd5b-1e98c03df11d
```

## Prerequisites

### 1. Docker
Must be installed and running. The skill will attempt to start Docker Desktop if it's not running.

### 2. AWS Credentials via Leapp

We use [Leapp](https://www.leapp.cloud/) to manage AWS credentials securely.

**Setup:**
1. Open Leapp application
2. Start a session for the Fence AWS account (e.g., "fence-prod")
3. Click the play button to activate the session
4. Credentials will be automatically populated in `~/.aws/credentials`

**Note:** Session tokens expire after ~1 hour. If queries fail with auth errors, restart the Leapp session.

## Features

- **Auto-routing**: Automatically selects the right log group based on keywords
- **Formatted output**: Parses JSON logs into readable format
- **Error analysis**: Extracts and displays stack traces
- **Source code tracing**: Maps errors to local source files in fence-backend
- **Fix suggestions**: Proposes code fixes when errors are found
- **Timeline view**: Shows events in chronological order

## Supported Log Groups

| Type | Keywords | Log Group |
|------|----------|-----------|
| API | api, request, asset, contract, 500, 400 | `/aws/fargate/api/prod` |
| Assets | assets processor, intake | `/aws/fargate/assets_processor/prod` |
| Metrics | metric, calculation, borrow base | Lambda metric-calculator |
| Webhooks | webhook, bridge, monerium | Lambda webhook processors |
| Workflows | long running, action | `/aws/fargate/long-running-service/prod` |
| Step Functions | deposit, withdrawal, repayment | Step Function logs |

## Output

The skill generates a structured investigation report including:

- **Timeline** of events
- **Error details** with full stack traces
- **Source code location** mapped to local fence-backend paths
- **Root cause analysis** explaining what went wrong
- **Suggested fixes** with code snippets
- **Recommended actions** checklist

## Example Output

```markdown
## CloudWatch Investigation: 57af6a49-74d2-41cf-bd5b-1e98c03df11d

### Errors Found

**Error:** TypeError: '<=' not supported between instances of 'str' and 'NoneType'
**File:** source/fence-core-lib/fence/core_lib/deals/mta/services/insights.py:923

### Root Cause Analysis

**What happened:** Asset sector_code comparison failed because one asset has sector_code = None

**Affected code:**
```python
if sector_range["min_cnae"] <= asset.sector_code <= sector_range["max_cnae"]:
```

### Suggested Fix

```python
# Change:
if sector_range["min_cnae"] <= asset.sector_code <= sector_range["max_cnae"]:

# To:
if asset.sector_code is not None and sector_range["min_cnae"] <= asset.sector_code <= sector_range["max_cnae"]:
```
```

## Path Mapping

The skill automatically maps deployed paths to local source paths:

| Deployed | Local |
|----------|-------|
| `/code/.venv/.../fence/core_lib/` | `source/fence-core-lib/fence/core_lib/` |
| `/code/app/` | `source/app/` |
