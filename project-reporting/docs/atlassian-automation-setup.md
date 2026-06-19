# Atlassian automation setup (manual)

Complete these steps in Atlassian Cloud after the GitHub workflow is installed. The automation rule and ROVO agent are project-agnostic — every project-specific value flows in through the webhook payload, so you do NOT edit the rule or agent prompt per project.

## 1. Create incoming webhook rule

1. Open **Jira** → **Project settings** → **Automation** (or global Automation).
2. Create a rule with trigger: **Incoming webhook**.
3. Copy the webhook URL — this becomes GitHub secret `ROVO_AUTOMATION_WEBHOOK_URL`.
4. If Atlassian provides a webhook token, store it as `ROVO_AUTOMATION_WEBHOOK_TOKEN`.

## 2. Parse the signal payload

The GitHub workflow sends JSON:

```json
{
  "report_date": "YYYY-MM-DD",
  "run_time_utc": "ISO-8601",
  "project_key": "ABC",
  "slack_channel": "#channel-name",
  "github_repo": "owner/repo",
  "jira_domain": "codeandtheory.atlassian.net",
  "timezone": "Asia/Kolkata",
  "start_hour": 9,
  "end_hour": 18,
  "request_source": "github_actions_daily_schedule",
  "prompt_hint": "Same-day crisp executive end-of-day status report..."
}
```

All project-specific values live in this payload. The automation rule and the ROVO agent reference them as `{{webhookData.<field>}}`.

## 3. Invoke ROVO agent

Add an action to run the ROVO agent (paste the prompt from [`rovo-agent-prompt.md`](./rovo-output-contract.md) once — it stays the same across every project). Pass these payload fields through:

- Jira project: `{{webhookData.project_key}}`
- Jira domain (for ticket URLs): `{{webhookData.jira_domain}}`
- GitHub repo: `{{webhookData.github_repo}}`
- Timezone: `{{webhookData.timezone}}`
- Window: `{{webhookData.start_hour}}:00` to `{{webhookData.end_hour}}:00` in `{{webhookData.timezone}}` on `{{webhookData.report_date}}`
- Prompt context: `{{webhookData.prompt_hint}}`

## 4. Map ROVO output to Slack

Format output per [rovo-output-contract.md](./rovo-output-contract.md):

- `report_header`, `executive_summary`, `today_completed`, `in_progress`
- `risks_blockers`, `next_24h`, `channel_targets`, `source_timestamp_utc`

Post to the Slack channel from `{{webhookData.slack_channel}}` — do not hardcode a channel.

## 5. Optional Confluence append

Use the Confluence template in `rovo-output-contract.md` to append an EOD status page.

## 6. Test

1. Set GitHub secrets and variables in the project repo (see [`github-secrets-vars.md`](./github-secrets-vars.md)).
2. Run **Daily ROVO Report Orchestrator** via `workflow_dispatch` in GitHub Actions, optionally overriding `timezone`, `start_hour`, `end_hour`, `project_key`, or `slack_channel`.
3. Confirm the automation rule fires and Slack receives the report.

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Workflow fails with "Missing required configuration" | Set the listed GitHub Actions variables, or pass them via `workflow_dispatch` inputs |
| Workflow fails on `start_hour`/`end_hour` validation | Ensure both are integers in `[0,23]` and `start_hour < end_hour` |
| Webhook not received | Verify `ROVO_AUTOMATION_WEBHOOK_URL` secret; check workflow run logs |
| 401/403 on webhook | Set `ROVO_AUTOMATION_WEBHOOK_TOKEN` if required by your rule |
| Wrong Jira project | Update `ROVO_TARGET_PROJECT_KEY` variable in GitHub, or pass `project_key` via `workflow_dispatch` |
| Wrong Slack channel | Update `ROVO_TARGET_SLACK_CHANNEL` variable in GitHub, or pass `slack_channel` via `workflow_dispatch` |
| Wrong timezone or window | Update `ROVO_REPORT_TIMEZONE` / `ROVO_REPORT_START_HOUR` / `ROVO_REPORT_END_HOUR` variables, or pass overrides via `workflow_dispatch` |
