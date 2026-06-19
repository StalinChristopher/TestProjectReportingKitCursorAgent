# Standard ROVO output contract

## Inputs from webhook

The ROVO agent reads every project-specific value from the incoming webhook payload sent by the `Daily ROVO Report Orchestrator` workflow. None of these are hardcoded in the prompt:

| Field | Type | Purpose |
|-------|------|---------|
| `project_key` | string | Jira project to query |
| `slack_channel` | string | Slack destination (includes leading `#`) |
| `github_repo` | string | `owner/repo` to scan for PR / branch activity |
| `jira_domain` | string | Domain used to build `https://<domain>/browse/<KEY>` URLs |
| `timezone` | string | IANA timezone for window resolution |
| `start_hour` | integer 0-23 | Window start hour (24-hour clock, local to `timezone`) |
| `end_hour` | integer 0-23 | Window end hour (must be greater than `start_hour`) |
| `report_date` | string `YYYY-MM-DD` | Date the report covers |
| `run_time_utc` | string ISO-8601 | When the scheduler triggered |
| `prompt_hint` | string | Narrative hint, informational only |

See [rovo-agent-prompt.md](./rovo-agent-prompt.md) for the canonical, project-agnostic agent prompt.

## Output structure

Use this structure so Slack and Confluence stay consistent across teams:

- `report_header`: `EOD Status - <project_key> - <YYYY-MM-DD>`
- `executive_summary`: 3-5 lines, decision-ready and non-technical
- `today_completed`: bullet list of major completed items
- `in_progress`: bullet list of meaningful in-flight work
- `risks_blockers`: explicit blocker/risk bullets with owner and next step
- `next_24h`: bullet list of planned next actions
- `channel_targets`: Slack channel(s) and Confluence destination
- `source_timestamp_utc`: report generation timestamp in UTC

## Paste-ready Confluence block template

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
