# GitHub Actions configuration (ROVO mode)

The `Daily ROVO Report Orchestrator` workflow resolves every project-specific value with this precedence:

`workflow_dispatch input` > GitHub Actions variable > **fail the run** (no defaults).

If any required field is unset for a scheduled run, the workflow fails fast and prints the missing keys.

## Required secrets

| Name | Purpose |
|------|---------|
| `ROVO_AUTOMATION_WEBHOOK_URL` | Incoming webhook URL for the Atlassian automation rule that invokes the ROVO agent |

## Optional secrets

| Name | Purpose |
|------|---------|
| `ROVO_AUTOMATION_WEBHOOK_TOKEN` | Value sent as `X-Automation-Webhook-Token` header if the automation rule requires it |

## Required variables

| Name | Example | Purpose |
|------|---------|---------|
| `ROVO_TARGET_PROJECT_KEY` | `ABC` | Jira project key |
| `ROVO_TARGET_SLACK_CHANNEL` | `#abc-eod-status` | Slack channel for delivery |
| `ROVO_REPORT_TIMEZONE` | `Asia/Kolkata` | IANA timezone for the reporting window |
| `ROVO_REPORT_START_HOUR` | `9` | Window start hour, 24-hour integer 0-23 |
| `ROVO_REPORT_END_HOUR` | `18` | Window end hour, 24-hour integer 0-23, must be greater than start |
| `ROVO_JIRA_DOMAIN` | `codeandtheory.atlassian.net` | Atlassian domain used to build ticket URLs |

## Optional variables

| Name | Default | Purpose |
|------|---------|---------|
| `ROVO_GITHUB_REPO` | `${{ github.repository }}` | Override the repo scanned for PR / branch activity |

## workflow_dispatch overrides

The following inputs override the matching variables for a single manual run, leaving repo settings untouched:

- `project_key`
- `slack_channel`
- `timezone`
- `start_hour`
- `end_hour`

## Quick setup with `gh`

```bash
gh secret set   ROVO_AUTOMATION_WEBHOOK_URL --repo "<owner>/<repo>" --body "<webhook-url>"
gh variable set ROVO_TARGET_PROJECT_KEY     --repo "<owner>/<repo>" --body "ABC"
gh variable set ROVO_TARGET_SLACK_CHANNEL   --repo "<owner>/<repo>" --body "#abc-eod-status"
gh variable set ROVO_REPORT_TIMEZONE        --repo "<owner>/<repo>" --body "Asia/Kolkata"
gh variable set ROVO_REPORT_START_HOUR      --repo "<owner>/<repo>" --body "9"
gh variable set ROVO_REPORT_END_HOUR        --repo "<owner>/<repo>" --body "18"
gh variable set ROVO_JIRA_DOMAIN            --repo "<owner>/<repo>" --body "codeandtheory.atlassian.net"
```

Never commit secret values to the repository.
