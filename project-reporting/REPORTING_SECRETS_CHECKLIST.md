# Reporting secrets & variables checklist

Project-specific values for **ROC** → `#reporting-work-temp`. Set these in GitHub repo **Settings → Secrets and variables → Actions** only — never commit secret values.

## Required secrets

| Name | Status | Notes |
|------|--------|-------|
| `ROVO_AUTOMATION_WEBHOOK_URL` | ☐ Not set | Incoming webhook URL from Atlassian Automation (see [atlassian-automation-setup.md](./docs/atlassian-automation-setup.md)) |

## Optional secrets

| Name | Status | Notes |
|------|--------|-------|
| `ROVO_AUTOMATION_WEBHOOK_TOKEN` | ☐ Not set | Only if your automation rule requires `X-Automation-Webhook-Token` |

## Required variables

| Name | Value | Status |
|------|-------|--------|
| `ROVO_TARGET_PROJECT_KEY` | `ROC` | ☐ Not set |
| `ROVO_TARGET_SLACK_CHANNEL` | `#reporting-work-temp` | ☐ Not set |
| `ROVO_REPORT_TIMEZONE` | `Asia/Kolkata` | ☐ Not set |
| `ROVO_REPORT_START_HOUR` | `9` | ☐ Not set |
| `ROVO_REPORT_END_HOUR` | `18` | ☐ Not set |
| `ROVO_JIRA_DOMAIN` | `codeandtheory.atlassian.net` | ☐ Not set |

## Optional variables

| Name | Default | Notes |
|------|---------|-------|
| `ROVO_GITHUB_REPO` | `StalinChristopher/TestProjectReportingKitCursorAgent` | Override only if scanning a different repo for PR activity |

## Quick setup with `gh`

After you have the Atlassian webhook URL:

```bash
REPO="StalinChristopher/TestProjectReportingKitCursorAgent"

gh secret   set ROVO_AUTOMATION_WEBHOOK_URL --repo "$REPO" --body "<webhook-url>"
gh variable set ROVO_TARGET_PROJECT_KEY     --repo "$REPO" --body "ROC"
gh variable set ROVO_TARGET_SLACK_CHANNEL   --repo "$REPO" --body "#reporting-work-temp"
gh variable set ROVO_REPORT_TIMEZONE        --repo "$REPO" --body "Asia/Kolkata"
gh variable set ROVO_REPORT_START_HOUR      --repo "$REPO" --body "9"
gh variable set ROVO_REPORT_END_HOUR        --repo "$REPO" --body "18"
gh variable set ROVO_JIRA_DOMAIN            --repo "$REPO" --body "codeandtheory.atlassian.net"
```

## Test

1. Complete [Atlassian automation setup](./docs/atlassian-automation-setup.md).
2. Set secrets and variables above in GitHub.
3. Run **Daily Project Report** via `workflow_dispatch` in GitHub Actions.
4. Confirm Slack channel `#reporting-work-temp` receives the report.

See also [github-secrets-vars.md](./docs/github-secrets-vars.md).
