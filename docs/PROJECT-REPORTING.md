# Project Reporting

This repo includes automated **daily end-of-day (EOD) status reports** that pull from Jira, summarize activity with Atlassian ROVO, and deliver to Slack. GitHub Actions acts as the **scheduler and signal layer** only — report generation and delivery happen in Atlassian Automation.

Related docs in this repo:

- [CT Code Metrics](./CT-METRICS.md) — AI attribution and template-vs-custom tracking
- [Expo App Overview](./EXPO-APP.md) — the React Native application this repo ships

Upstream reference: [codeandtheory/ReportingAgent](https://github.com/codeandtheory/ReportingAgent)

---

## Table of contents

1. [Architecture overview](#architecture-overview)
2. [How it works](#how-it-works)
3. [Configuration in this repo](#configuration-in-this-repo)
4. [GitHub Actions workflow](#github-actions-workflow)
5. [Running the pipeline](#running-the-pipeline)
6. [Atlassian Automation setup](#atlassian-automation-setup)
7. [ROVO output contract](#rovo-output-contract)
8. [Secrets and variables checklist](#secrets-and-variables-checklist)
9. [Troubleshooting](#troubleshooting)

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  GitHub Actions — .github/workflows/daily_project_report.yml            │
│                                                                         │
│  Trigger: cron (Mon–Fri) or workflow_dispatch                           │
│       │                                                                 │
│       ▼                                                                 │
│  Resolve config (vars / manual inputs)                                  │
│       │                                                                 │
│       ▼                                                                 │
│  POST webhook payload ──────────────────────────────────────────────┐   │
└─────────────────────────────────────────────────────────────────────│───┘
                                                                      │
                                                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Atlassian Automation (Jira Cloud)                                      │
│                                                                         │
│  Incoming webhook trigger                                               │
│       │                                                                 │
│       ▼                                                                 │
│  Invoke ROVO agent with payload fields (project, window, repo, …)      │
│       │                                                                 │
│       ▼                                                                 │
│  Format output → post to Slack channel                                  │
│       │                                                                 │
│       └── (optional) append to Confluence EOD page                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**ROVO-first design:** The automation rule and ROVO agent prompt are **project-agnostic**. Every project-specific value (Jira key, Slack channel, timezone, reporting window) flows through the webhook payload from GitHub. You do not edit the Atlassian rule per repo.

---

## How it works

### Step 1 — Schedule fires in GitHub

The workflow runs on a cron schedule (weekdays) or when triggered manually. It does **not** call Jira or Slack directly.

### Step 2 — Resolve configuration

The workflow resolves each field with this precedence:

```
workflow_dispatch input  >  GitHub Actions variable  >  fail the run
```

There are no silent defaults for scheduled runs. If a required variable is missing, the workflow fails fast and lists the missing keys in the log.

### Step 3 — Signal Atlassian

The workflow POSTs a JSON payload to `ROVO_AUTOMATION_WEBHOOK_URL`. The payload includes:

| Field | Purpose |
|---|---|
| `project_key` | Jira project to query |
| `slack_channel` | Slack destination (with `#`) |
| `github_repo` | `owner/repo` for PR / branch activity |
| `jira_domain` | Domain for ticket URLs |
| `timezone` | IANA timezone for the reporting window |
| `start_hour` / `end_hour` | 24-hour window bounds (local to timezone) |
| `report_date` | Date the report covers (`YYYY-MM-DD`) |
| `run_time_utc` | When the scheduler triggered |
| `prompt_hint` | Narrative context for the ROVO agent |

Both snake_case and camelCase keys are sent for compatibility with Atlassian Studio webhook parsing.

### Step 4 — ROVO generates the report

The Atlassian automation rule receives the webhook, invokes the ROVO agent with the payload fields, and formats the response per the [output contract](#rovo-output-contract).

### Step 5 — Deliver to Slack (and optionally Confluence)

The automation posts to `{{webhookData.slack_channel}}` — never a hardcoded channel. An optional Confluence append uses the template in `project-reporting/docs/rovo-output-contract.md`.

---

## Configuration in this repo

### `project-reporting/reporting.config.json`

Project-specific values captured at setup time:

```json
{
  "jira_project_key": "ROC",
  "slack_channel": "#reporting-work-temp",
  "jira_domain": "codeandtheory.atlassian.net",
  "report_timezone": "Asia/Kolkata",
  "report_start_hour": 9,
  "report_end_hour": 18,
  "schedule_cron": "30 3 * * 1-5",
  "repo_url": "https://github.com/StalinChristopher/TestProjectReportingKitCursorAgent.git",
  "template_commit": "b2275ba"
}
```

This file is a **reference record** for humans and tooling. The workflow reads GitHub **secrets and variables**, not this JSON directly. Keep them in sync when you change project settings.

| Setting | Value | Meaning |
|---|---|---|
| Jira project | `ROC` | Tickets queried for the EOD report |
| Slack channel | `#reporting-work-temp` | Where reports are delivered |
| Timezone | `Asia/Kolkata` | Window is 9:00–18:00 IST |
| Schedule | `30 3 * * 1-5` | Mon–Fri at 03:30 UTC (9:00 AM IST) |

### Files added by project reporting setup

| Path | Purpose |
|---|---|
| `.github/workflows/daily_project_report.yml` | Scheduler / webhook signal workflow |
| `project-reporting/reporting.config.json` | Project config reference |
| `project-reporting/REPORTING_SECRETS_CHECKLIST.md` | Quick secrets checklist |
| `project-reporting/docs/atlassian-automation-setup.md` | Atlassian-side setup steps |
| `project-reporting/docs/github-secrets-vars.md` | GitHub secrets and variables reference |
| `project-reporting/docs/rovo-output-contract.md` | ROVO output structure for Slack / Confluence |
| `.cursor/skills/project-reporting-kit/SKILL.md` | Cursor skill to re-run or refresh setup |

---

## GitHub Actions workflow

**File:** `.github/workflows/daily_project_report.yml`

### Triggers

| Event | When it runs |
|---|---|
| `schedule` | Mon–Fri at `30 3 * * *` UTC (9:00 AM IST with current config) |
| `workflow_dispatch` | Manual run from the Actions tab, with optional overrides |

### Job: `rovo-signal`

1. **Checkout** — checks out the repo (no build artifacts needed)
2. **Resolve and validate inputs** — merges dispatch inputs with GitHub variables; validates hour range
3. **Signal Atlassian automation webhook** — POSTs the payload; fails if the webhook returns a non-2xx status
4. **Write run summary** — adds a summary table to the workflow run page

If `ROVO_AUTOMATION_WEBHOOK_URL` is not set, the webhook step is skipped (workflow still completes with a summary noting webhook is not configured).

---

## Running the pipeline

### Option A — Wait for the schedule (automatic)

After secrets and variables are configured, the workflow runs automatically on weekdays at the configured cron time.

### Option B — Manual dispatch (recommended for testing)

1. Open **GitHub → Actions → Daily Project Report**
2. Click **Run workflow**
3. Optionally override:
   - `project_key` — Jira project (default: `ROVO_TARGET_PROJECT_KEY`)
   - `slack_channel` — Slack channel (default: `ROVO_TARGET_SLACK_CHANNEL`)
   - `timezone` — IANA timezone (default: `ROVO_REPORT_TIMEZONE`)
   - `start_hour` / `end_hour` — reporting window (defaults from variables)
4. Click **Run workflow**

Manual overrides apply to **that run only** — repo variables are unchanged.

### Option C — Trigger via GitHub CLI

```bash
gh workflow run daily_project_report.yml \
  --repo StalinChristopher/TestProjectReportingKitCursorAgent

# With overrides:
gh workflow run daily_project_report.yml \
  --repo StalinChristopher/TestProjectReportingKitCursorAgent \
  -f project_key=ROC \
  -f slack_channel="#reporting-work-temp" \
  -f timezone=Asia/Kolkata \
  -f start_hour=9 \
  -f end_hour=18
```

### Verifying a run

1. Open the workflow run in GitHub Actions
2. Check the **Resolve and validate inputs** step log for resolved values
3. Check **Signal Atlassian automation webhook** for HTTP status and response body
4. Read the **Summary** tab for the orchestration overview
5. Confirm the Slack channel received the report

---

## Atlassian Automation setup

Complete these steps once in Atlassian Cloud. The rule is shared across projects — only the webhook payload changes per repo.

### 1. Create incoming webhook rule

1. Open **Jira → Project settings → Automation** (or global Automation)
2. Create a rule with trigger: **Incoming webhook**
3. Copy the webhook URL → GitHub secret `ROVO_AUTOMATION_WEBHOOK_URL`
4. If Atlassian provides a token → GitHub secret `ROVO_AUTOMATION_WEBHOOK_TOKEN`

### 2. Parse the signal payload

Reference fields as `{{webhookData.project_key}}`, `{{webhookData.slack_channel}}`, etc.

Example payload the workflow sends:

```json
{
  "report_date": "2026-07-02",
  "run_time_utc": "2026-07-02T03:30:00Z",
  "project_key": "ROC",
  "slack_channel": "#reporting-work-temp",
  "github_repo": "StalinChristopher/TestProjectReportingKitCursorAgent",
  "jira_domain": "codeandtheory.atlassian.net",
  "timezone": "Asia/Kolkata",
  "start_hour": 9,
  "end_hour": 18,
  "request_source": "github_actions_daily_schedule",
  "prompt_hint": "Same-day crisp executive end-of-day status report..."
}
```

### 3. Invoke ROVO agent

Add an action to run the ROVO agent. Pass payload fields through — do not hardcode project keys or channels. The canonical agent prompt lives in `project-reporting/docs/rovo-output-contract.md`.

### 4. Map output to Slack

Format ROVO output per the [output contract](#rovo-output-contract) and post to `{{webhookData.slack_channel}}`.

### 5. Test end-to-end

1. Set GitHub secrets and variables (see below)
2. Run **Daily Project Report** via `workflow_dispatch`
3. Confirm Slack channel `#reporting-work-temp` receives the report

Full step-by-step: [`project-reporting/docs/atlassian-automation-setup.md`](../project-reporting/docs/atlassian-automation-setup.md)

---

## ROVO output contract

The ROVO agent should return structured output so Slack and Confluence stay consistent:

| Field | Description |
|---|---|
| `report_header` | `EOD Status - <project_key> - <YYYY-MM-DD>` |
| `executive_summary` | 3–5 lines, decision-ready, non-technical |
| `today_completed` | Bullet list of major completed items |
| `in_progress` | Bullet list of in-flight work |
| `risks_blockers` | Blockers/risks with owner and next step |
| `next_24h` | Planned next actions |
| `channel_targets` | Slack channel(s) and Confluence destination |
| `source_timestamp_utc` | Report generation timestamp (UTC) |

Confluence paste template:

```md
h2. EOD Status - <project_key> - <YYYY-MM-DD>

*Executive Summary*
<3-5 line crisp summary>

*Completed Today*
- ...

*In Progress*
- ...

*Risks / Blockers*
- ...

*Next 24 Hours*
- ...
```

Full contract: [`project-reporting/docs/rovo-output-contract.md`](../project-reporting/docs/rovo-output-contract.md)

---

## Secrets and variables checklist

Set in **GitHub → Settings → Secrets and variables → Actions**. Never commit secret values.

### Required secrets

| Name | Purpose |
|---|---|
| `ROVO_AUTOMATION_WEBHOOK_URL` | Incoming webhook URL from Atlassian Automation |

### Optional secrets

| Name | Purpose |
|---|---|
| `ROVO_AUTOMATION_WEBHOOK_TOKEN` | Sent as `X-Automation-Webhook-Token` if the rule requires it |

### Required variables

| Name | This repo's value |
|---|---|
| `ROVO_TARGET_PROJECT_KEY` | `ROC` |
| `ROVO_TARGET_SLACK_CHANNEL` | `#reporting-work-temp` |
| `ROVO_REPORT_TIMEZONE` | `Asia/Kolkata` |
| `ROVO_REPORT_START_HOUR` | `9` |
| `ROVO_REPORT_END_HOUR` | `18` |
| `ROVO_JIRA_DOMAIN` | `codeandtheory.atlassian.net` |

### Optional variables

| Name | Default | Purpose |
|---|---|---|
| `ROVO_GITHUB_REPO` | Current repo | Override if scanning a different repo for PR activity |

### Quick setup with `gh`

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

Printable checklist: [`project-reporting/REPORTING_SECRETS_CHECKLIST.md`](../project-reporting/REPORTING_SECRETS_CHECKLIST.md)

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Missing required configuration` | GitHub variables not set | Set all required variables, or pass via `workflow_dispatch` |
| `start_hour must be less than end_hour` | Invalid window | Use integers 0–23 with `start_hour < end_hour` |
| Webhook step skipped | `ROVO_AUTOMATION_WEBHOOK_URL` unset | Set the secret in GitHub |
| Webhook returns 401/403 | Missing or wrong token | Set `ROVO_AUTOMATION_WEBHOOK_TOKEN` |
| Workflow succeeds but no Slack message | Atlassian rule misconfigured | Check automation run history in Jira; verify ROVO action and Slack step |
| Wrong Jira project | Stale variable | Update `ROVO_TARGET_PROJECT_KEY` or override via dispatch |
| Wrong Slack channel | Stale variable | Update `ROVO_TARGET_SLACK_CHANNEL` or override via dispatch |
| Wrong timezone/window | Stale variables | Update `ROVO_REPORT_*` variables or override via dispatch |

### Refreshing the workflow template

To pull the latest workflow from the ReportingAgent template, re-run setup in Cursor:

> Set up project reporting

Or follow `.cursor/skills/project-reporting-kit/SKILL.md`.

---

## Security notes

- Never commit webhook URLs, tokens, or API keys
- GitHub secrets are encrypted; variables are visible to repo collaborators with Actions access
- The workflow only **signals** Atlassian — it does not store Jira or Slack credentials in GitHub
