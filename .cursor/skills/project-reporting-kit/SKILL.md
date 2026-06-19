---
name: project-reporting-kit
description: "Set up project reporting. Apply the project reporting kit. Configure daily Jira Slack reports. Triggered by \"Set up project reporting \". Clones or updates CursorReactNativeAgents, installs project-reporting kit, clones ReportingAgent template, applies workflow and docs."
metadata:
  kitGitRepo: git@github.com:codeandtheory/CursorReactNativeAgents.git
  kitGitHttps: https://github.com/codeandtheory/CursorReactNativeAgents.git
  kitGitBranch: main
  kitCacheDir: ~/.cache/CursorReactNativeAgents
  templateGitUrl: https://github.com/codeandtheory/ReportingAgent.git
  templateGitSsh: git@github.com:codeandtheory/ReportingAgent.git
  templateBranch: main
  templateCacheDir: ~/.cache/ReportingAgent
---

# Project Reporting Kit

User says: **Set up project reporting** → refresh kit from GitHub first, then clone ReportingAgent template and apply reporting files in one session.

Repo URL, branch, and template source: read `project-reporting/kit.config.json` in the cache after clone.

## Hard constraints

- **Phase 1 before any user question** (including first run).
- **Never** ask for a manual agents-repo path.
- **Shell-first** — Use install scripts; do not hand-copy template files.
- **Never** commit webhook URLs, tokens, or API keys.
- **Never** require `package.json`, Node, Python, or any app runtime.
- **Always** refresh ReportingAgent template on kit run (fetch latest `main`).

## Phase 0 — Workspace

Confirm `test -d .git` at workspace root. `APP_ROOT` = that directory.

If the user opened a monorepo subfolder, **AskQuestion**: install at repo root (default) or subpath?

Warn (non-fatal) if `origin` remote is missing or not GitHub-hosted.

## Phase 1 — Clone + install kit (no user input)

```bash
cd "$APP_ROOT"
CACHE="${HOME}/.cache/CursorReactNativeAgents"

if [[ ! -f "$CACHE/project-reporting/install/run-project-reporting-setup.sh" ]]; then
  git clone --depth 1 https://github.com/codeandtheory/CursorReactNativeAgents.git "$CACHE" \
    || git clone --depth 1 git@github.com:codeandtheory/CursorReactNativeAgents.git "$CACHE"
fi

bash "$CACHE/project-reporting/install/run-project-reporting-setup.sh" "$APP_ROOT"
```

Always refresh from GitHub — do not skip because the kit already exists in the project.

## Phase 2 — Resolve TEMPLATE_ROOT

Read `templateGitUrl` and `templateBranch` from `$CACHE/project-reporting/kit.config.json`. Do **not** ask the user for this URL unless clone fails.

```bash
TEMPLATE_GIT_URL="https://github.com/codeandtheory/ReportingAgent.git"
TEMPLATE_BRANCH="main"
TEMPLATE_CACHE="${HOME}/.cache/ReportingAgent"
```

**Default:** shallow-clone or fetch ReportingAgent into `$TEMPLATE_CACHE`. Request **network** permissions if the sandbox blocks `git`.

```bash
export TEMPLATE_ROOT="${TEMPLATE_CACHE}"
bash "$CACHE/project-reporting/install/apply-reporting-template.sh" "$APP_ROOT"
# First run without --force; script skips existing dest files
```

Preflight: `test -f "$TEMPLATE_ROOT/SETUP_PROJECT_REPORTING.md"`

On clone failure, stop with actionable guidance: private repo may need SSH (`git@github.com:codeandtheory/ReportingAgent.git`), `gh auth login`, or a configured credential helper.

**Override only when explicit:** If the user message contains `PATH:` or `TEMPLATE_ROOT=`, skip clone and use that path. Confirm `SETUP_PROJECT_REPORTING.md` exists.

## Phase 3 — Apply template (overwrite prompt)

If `.github/workflows/daily_project_report.yml` already exists and Phase 2 skipped it:

**AskQuestion:** Overwrite workflow and docs from latest ReportingAgent template?

If yes:

```bash
export TEMPLATE_ROOT="${TEMPLATE_CACHE}"
bash "$CACHE/project-reporting/install/apply-reporting-template.sh" "$APP_ROOT" --force
```

Capture `TEMPLATE_COMMIT` from script output for handoff.

## Phase 4 — Project configuration

**AskQuestion** (collect ALL of the following — these become GitHub Actions variables; the ROVO agent reads them from the webhook payload, no agent edits required):

- Jira project key (e.g. `ABC`) → `ROVO_TARGET_PROJECT_KEY`
- Slack channel including `#` (e.g. `#team-eod-status`) → `ROVO_TARGET_SLACK_CHANNEL`
- Jira domain (e.g. `codeandtheory.atlassian.net`) → `ROVO_JIRA_DOMAIN`
- Reporting timezone, IANA (e.g. `Asia/Kolkata`, `America/New_York`, `UTC`) → `ROVO_REPORT_TIMEZONE`
- Window start hour, 24h integer 0-23 (e.g. `9`) → `ROVO_REPORT_START_HOUR`
- Window end hour, 24h integer 0-23 and greater than start (e.g. `18`) → `ROVO_REPORT_END_HOUR`
- Schedule timezone for cron: IST default (Mon–Fri 9:00 AM IST / `30 3 * * 1-5`) / US Eastern / custom cron

Validate locally: `start_hour` and `end_hour` are integers in `[0,23]` and `start_hour < end_hour`. The workflow re-validates and fails fast if these are missing or invalid.

If non-default schedule, patch cron in `.github/workflows/daily_report.yml`.

## Phase 5 — Write reporting.config.json

Write `project-reporting/reporting.config.json` (non-secret metadata only):

```json
{
  "jira_project_key": "<from user>",
  "slack_channel": "<from user>",
  "jira_domain": "<from user>",
  "report_timezone": "<from user>",
  "report_start_hour": <0-23>,
  "report_end_hour": <0-23>,
  "schedule_cron": "<cron expression>",
  "team_name": "<optional>",
  "repo_url": "<origin url if available>",
  "template_commit": "<TEMPLATE_COMMIT>"
}
```

GitHub variables remain the runtime source of truth for the workflow.

## Phase 6 — Generate secrets checklist

Create `project-reporting/REPORTING_SECRETS_CHECKLIST.md` from `project-reporting/docs/github-secrets-vars.md` plus user-specific values (project key, channel). Remind: set values in GitHub repo Settings only — do not commit secrets.

## Phase 7 — Atlassian ROVO setup

Walk through `project-reporting/docs/atlassian-automation-setup.md` (manual checklist):

1. Create Atlassian Automation rule (incoming webhook trigger)
2. Copy webhook URL → GitHub secret `ROVO_AUTOMATION_WEBHOOK_URL`
3. Configure ROVO agent with Jira project scope
4. Map ROVO output to Slack per `rovo-output-contract.md`
5. Optional Confluence page append
6. Test with `workflow_dispatch`

## Phase 8 — GitHub configuration

If `gh` CLI is available and authenticated against the project repo, offer to push variables and the webhook secret automatically. Otherwise print the commands for the user to run.

Required secrets:

- `ROVO_AUTOMATION_WEBHOOK_URL`
- `ROVO_AUTOMATION_WEBHOOK_TOKEN` (optional)

Required variables (all must be set — the workflow fails fast if any are missing):

- `ROVO_TARGET_PROJECT_KEY`
- `ROVO_TARGET_SLACK_CHANNEL`
- `ROVO_REPORT_TIMEZONE`
- `ROVO_REPORT_START_HOUR`
- `ROVO_REPORT_END_HOUR`
- `ROVO_JIRA_DOMAIN`

Optional variable:

- `ROVO_GITHUB_REPO` (defaults to `${{ github.repository }}`)

Emit (and optionally execute) these commands using values collected in Phase 4:

```bash
REPO="<owner>/<repo>"
gh secret   set ROVO_AUTOMATION_WEBHOOK_URL --repo "$REPO" --body "<webhook-url>"
gh variable set ROVO_TARGET_PROJECT_KEY     --repo "$REPO" --body "<project_key>"
gh variable set ROVO_TARGET_SLACK_CHANNEL   --repo "$REPO" --body "<slack_channel>"
gh variable set ROVO_REPORT_TIMEZONE        --repo "$REPO" --body "<timezone>"
gh variable set ROVO_REPORT_START_HOUR      --repo "$REPO" --body "<start_hour>"
gh variable set ROVO_REPORT_END_HOUR        --repo "$REPO" --body "<end_hour>"
gh variable set ROVO_JIRA_DOMAIN            --repo "$REPO" --body "<jira_domain>"
```

Suggest running the workflow via **workflow_dispatch** to validate the signal. The dispatch form also accepts one-off overrides for `project_key`, `slack_channel`, `timezone`, `start_hour`, and `end_hour`.

## Phase 9 — Handoff

Include in handoff:

- `TEMPLATE_COMMIT` applied from ReportingAgent
- Files copied vs skipped
- Path to `REPORTING_SECRETS_CHECKLIST.md`
- Manual follow-ups: Atlassian automation, GitHub secrets/vars
- Re-run **Set up project reporting** to refresh from latest ReportingAgent `main`

Optional verification:

```bash
bash "$CACHE/project-reporting/install/verify-app-reporting-layout.sh" "$APP_ROOT"
```

## Optional user inputs (never required for template URL)

- `PATH:` or `TEMPLATE_ROOT=` — local ReportingAgent checkout override
- `TARGET_REMOTE_URL` — commit + push only if user explicitly requests
