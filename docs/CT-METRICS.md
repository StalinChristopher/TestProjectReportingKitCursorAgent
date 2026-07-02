# CT Code Metrics

This repo tracks two complementary code metrics:

| Tool | What it measures | Where it runs |
|---|---|---|
| **AI Attribution Hooks** | % of each commit that was AI-generated vs human-written | Local (Cursor / Claude Code + git) |
| **Template Code Metrics** | % of the repo that is still original template vs customized | GitHub Actions CI |

Both feed into a shared **DevLake → Grafana** dashboard so leadership can compare repos over time.

Upstream reference: [codeandtheory/ct-github-code-metrics](https://github.com/codeandtheory/ct-github-code-metrics)

---

## Table of contents

1. [Architecture overview](#architecture-overview)
2. [AI Attribution Hooks](#ai-attribution-hooks)
3. [Template Code Metrics](#template-code-metrics)
4. [CI pipeline (GitHub Actions)](#ci-pipeline-github-actions)
5. [Running metrics locally](#running-metrics-locally)
6. [Configuration in this repo](#configuration-in-this-repo)
7. [Required GitHub secrets](#required-github-secrets)
8. [Viewing results in Grafana](#viewing-results-in-grafana)
9. [Troubleshooting](#troubleshooting)

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Developer machine                                                      │
│                                                                         │
│  Cursor / Claude Code                                                   │
│       │ afterFileEdit / PostToolUse                                     │
│       ▼                                                                 │
│  snapshot_ai_output.py  ──▶  .cursor/ai_snapshots/  (git-ignored)      │
│                           ──▶  .claude/ai_snapshots/  (git-ignored)     │
│       │ stop / Stop                                                     │
│       ▼                                                                 │
│  install prepare-commit-msg git hook  (.git/hooks/ — local only)        │
│       │ git commit                                                      │
│       ▼                                                                 │
│  compute_attribution.py  ──▶  commit message stamped with AI %          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ push to main
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  GitHub Actions — .github/workflows/template-metrics.yml                │
│                                                                         │
│  1. checkout (full history)                                             │
│  2. download template-metrics.mjs + ai-attribution-metrics.mjs          │
│  3. compute template vs custom %  (+ optional semantic score)           │
│  4. parse commit messages for AI attribution blocks                     │
│  5. upload metrics.json + ai-attribution.json as artifacts              │
│  6. POST combined payload to DevLake webhook                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                          DevLake (MySQL)  ──▶  Grafana dashboard
```

---

## AI Attribution Hooks

### Purpose

Every time an AI assistant writes or edits a file, the system snapshots that output. At commit time, it compares the snapshot against what you actually staged, computes a per-file AI vs human percentage, and appends a summary to the commit message.

This captures the **true ratio** — including any edits you made after the AI finished — because attribution runs at commit time, not when the AI session ends.

### Three-stage local pipeline

```
┌─────────────┐       ┌─────────────┐       ┌──────────────┐
│  Stage 1    │       │  Stage 2    │       │  Stage 3     │
│  SNAPSHOT   │ ────▶ │  INSTALL    │ ────▶ │  ATTRIBUTE   │
│             │       │             │       │              │
│ Capture AI  │       │ Ensure git  │       │ Diff against │
│ output the  │       │ hook is in  │       │ staged code, │
│ instant it  │       │ place when  │       │ stamp commit │
│ writes      │       │ session ends│       │ message      │
└─────────────┘       └─────────────┘       └──────────────┘
```

#### Stage 1 — Snapshot AI output

| Tool | Trigger | Script |
|---|---|---|
| **Cursor** | `afterFileEdit` (agent + Tab completions) | `.cursor/hooks/snapshot_ai_output.py` |
| **Claude Code** | `PostToolUse` on `Write` / `Edit` / `MultiEdit` | `.claude/hooks/snapshot_ai_output.py` |

Each snapshot is a JSON file in `.cursor/ai_snapshots/` or `.claude/ai_snapshots/` containing the absolute file path, timestamp, session ID, and full file content. Snapshots are keyed by a hash of the path — repeat edits to the same file overwrite the previous snapshot.

#### Stage 2 — Install the git hook

| Tool | Trigger | Script |
|---|---|---|
| **Cursor** | `stop` | `.cursor/hooks/stop_install_git_hook.py` |
| **Claude Code** | `Stop` | `.claude/hooks/stop_generate_attribution.py` |

When an AI session ends and snapshots exist, the hook installer writes (or chains into) `.git/hooks/prepare-commit-msg`. This is **local to each clone** — not tracked by git. A new contributor must run at least one AI session before the git hook is installed on their machine.

The Cursor and Claude installers **coexist**: each reads only its own snapshot directory and appends its own summary block when relevant.

#### Stage 3 — Compute attribution at commit time

**Script:** `.cursor/hooks/compute_attribution.py` or `.claude/hooks/compute_attribution.py` (invoked from the shared git hook)

For every staged file:

1. Look up the AI snapshot (if one exists)
2. Read the file's current staged content
3. Use Python's `difflib.SequenceMatcher` to compare snapshot vs staged version
4. Count unchanged lines as **AI** and inserted/replaced lines as **human**
5. Append a formatted summary to the commit message

Files with no AI snapshot are counted as 100% human. Merge and squash commits are skipped. All failure paths exit with code 0 so hooks never block commits.

### Example commit message block

When a commit includes AI-touched files, this block is appended automatically:

```
─────────────────────────────────────────
🤖 AI Attribution Summary (Cursor)
─────────────────────────────────────────
  Overall  →  AI: 72%  |  Human: 28%
  Generated: 2026-07-02 14:30 UTC

  Per-file breakdown:
    src/screens/home/HomeScreen.tsx
      [████████░░]  AI 82% (147 lines)  /  Human 18% (32 lines)
─────────────────────────────────────────
```

### File structure

```
.claude/
├── settings.json                      ← Claude Code hook definitions
├── hooks/
│   ├── snapshot_ai_output.py
│   ├── stop_generate_attribution.py
│   └── compute_attribution.py
└── ai_snapshots/                      ← auto-generated, git-ignored

.cursor/
├── hooks.json                         ← Cursor hook definitions
├── hooks/
│   ├── snapshot_ai_output.py
│   ├── stop_install_git_hook.py
│   └── compute_attribution.py
└── ai_snapshots/                      ← auto-generated, git-ignored

.git/hooks/
└── prepare-commit-msg                 ← auto-installed locally, not in git
```

### Prerequisites (local)

- **Python 3** on `PATH` (hooks call `python3`)
- **Git**
- **Cursor** and/or **Claude Code** with hooks enabled (config is already in this repo)

### Important caveats

- **Snapshot overwrites:** Only the last AI edit per file is kept within a session.
- **Stale snapshots:** Old snapshots persist on disk. Clear `.cursor/ai_snapshots/` and `.claude/ai_snapshots/` periodically if you want a clean baseline.
- **Deleted lines aren't counted:** Percentages reflect the composition of the final file, not total editing effort.
- **Line-level granularity:** Changing one character on an AI-written line counts the whole line as human.
- **CI commits:** Commits made outside an AI session have no snapshots and produce no attribution block (correct behavior).

---

## Template Code Metrics

### Purpose

Measures what percentage of the current codebase came verbatim from a **baseline commit** (the original template scaffold) vs what has been added or modified since then.

For this React Native / Expo project, the baseline is the **root commit** — the first commit when the repo was generated from the template.

### How the score is computed

The script (`template-metrics.mjs`) runs `git diff --numstat` from the baseline commit to `HEAD`, excluding lockfiles, build artifacts, and generated paths.

For each tracked file at `HEAD`:

| File state | Template LOC | Custom LOC |
|---|---|---|
| Unchanged since baseline | all current lines | 0 |
| Modified since baseline | unchanged portion | added/changed lines |
| Newly added | 0 | all lines |
| Deleted since baseline | removed lines (churn metric) | — |

```
template_pct = template_unchanged / (template_unchanged + custom_added)
custom_pct   = custom_added       / (template_unchanged + custom_added)
```

### Semantic scoring (enabled in this repo)

When `ANTHROPIC_API_KEY` is set in CI, the script also sends diffs to Claude (`claude-haiku-4-5-20251001`) to classify changes as **substantive** (logic/behavior) vs **cosmetic** (formatting, renames, comments). The result appears as `template_pct_semantic` in the metrics JSON and as additional Grafana panels.

- Cost: ~1¢ per commit
- Skipped automatically when the secret is unset, the API fails, or the diff exceeds 10,000 lines
- Does not affect the deterministic score if it fails

---

## CI pipeline (GitHub Actions)

**Workflow file:** `.github/workflows/template-metrics.yml`

### Triggers

| Event | When it runs |
|---|---|
| `push` to `main` | Every push to the default branch |
| `pull_request` closed | Only when the PR is **merged** (not on close-without-merge) |
| `workflow_dispatch` | Manual run from the Actions tab |

### Pipeline steps

1. **Checkout** — full git history (`fetch-depth: 0`) is required for baseline diffs
2. **Download scripts** — fetches `template-metrics.mjs` and `ai-attribution-metrics.mjs` from the upstream repo using `CT_METRICS_TOKEN`
3. **Compute template metrics** — runs `node template-metrics.mjs > metrics.json`
4. **Compute AI attribution metrics** — parses commit messages between baseline and HEAD for attribution blocks; runs `node ai-attribution-metrics.mjs > ai-attribution.json`
5. **Upload artifacts** — saves both JSON files as `ct-metrics-<sha>` artifacts (retained by GitHub for 90 days by default)
6. **POST to DevLake** — merges both JSON files and sends a deployment webhook payload. Skips gracefully if `DEVLAKE_WEBHOOK_URL` or `DEVLAKE_BASIC_AUTH` are not set.

### Running the pipeline

#### Option A — Push to main (automatic)

After secrets are configured, merge or push to `main`. The workflow starts within seconds.

#### Option B — Manual dispatch (no code change needed)

1. Open **GitHub → Actions → Template Metrics**
2. Click **Run workflow**
3. Select the branch (usually `main`) and click **Run workflow**

Use this to verify secrets, re-run after fixing configuration, or backfill a data point without a new commit.

#### Option C — Re-run a failed job

1. Open the failed workflow run
2. Click **Re-run all jobs** or **Re-run failed jobs**

### Checking pipeline output

During a run, the workflow logs print summary lines:

```
=== Template Code Metrics ===
{ "template_pct": 45, "custom_pct": 55, "total_loc": 12400 }

=== AI Attribution Metrics ===
{ "ai_pct": 62, "human_pct": 38, "commits_with_attribution": 8, "commits_analyzed": 42 }
```

After the run completes:

1. Open the workflow run page
2. Scroll to **Artifacts**
3. Download `ct-metrics-<sha>` — contains `metrics.json` and `ai-attribution.json`

If DevLake secrets are set, the repo appears in Grafana within ~1 minute of a successful POST.

---

## Running metrics locally

Local runs are useful for validating configuration before pushing, or debugging unexpected percentages.

### Prerequisites

- Node.js 18+ (for the metrics scripts)
- `jq` (optional, for pretty-printing JSON)
- Full git history (`git fetch --unshallow` if you have a shallow clone)
- For semantic scoring locally: `ANTHROPIC_API_KEY` exported in your shell

### Download the scripts

```bash
curl -fsSL \
  -H "Authorization: token <your-github-pat>" \
  https://raw.githubusercontent.com/codeandtheory/ct-github-code-metrics/main/core/scripts/template-metrics.mjs \
  -o template-metrics.mjs

curl -fsSL \
  -H "Authorization: token <your-github-pat>" \
  https://raw.githubusercontent.com/codeandtheory/ct-github-code-metrics/main/core/scripts/ai-attribution-metrics.mjs \
  -o ai-attribution-metrics.mjs
```

Use a GitHub PAT with `repo` read access to `codeandtheory/ct-github-code-metrics`, or clone that repo and run the scripts from `core/scripts/`.

Add the downloaded scripts to `.gitignore` if you keep them in the repo root — they are fetched fresh in CI and should not be committed.

### Run template metrics

From the repo root (where `.template-provenance.json` lives):

```bash
node template-metrics.mjs | jq '.percentages, .counts'
```

Full output:

```bash
node template-metrics.mjs | jq .
```

With semantic scoring:

```bash
ANTHROPIC_API_KEY=sk-ant-... node template-metrics.mjs | jq '.percentages'
```

Expected output fields:

| Field | Meaning |
|---|---|
| `percentages.template_pct` | % of current LOC unchanged from baseline |
| `percentages.custom_pct` | % of current LOC added or modified |
| `percentages.template_pct_semantic` | Semantic-adjusted template % (null if semantic disabled) |
| `counts.current_total_loc` | Total lines counted at HEAD |
| `counts.template_unchanged_loc` | Lines identical to baseline |
| `counts.custom_added_loc` | Lines added or changed since baseline |

Sanity checks:

- Fresh template repo → `template_pct: 100`, `custom_pct: 0`
- Heavily customized repo → lower `template_pct`
- `current_total_loc: 0` → exclusions may be too aggressive; check `extra_excludes`

### Run AI attribution metrics

Parses commit messages from baseline to HEAD:

```bash
node ai-attribution-metrics.mjs | jq '.attribution'
```

If no commits contain attribution blocks yet (no AI-assisted commits since baseline), `ai_pct` and `human_pct` will be `null`. Make at least one commit using Cursor or Claude Code after the hooks are active.

### Test AI attribution hooks locally

1. Open this repo in Cursor (hooks in `.cursor/hooks.json` are already configured)
2. Ask the agent to edit a file
3. Confirm a snapshot appeared: `ls .cursor/ai_snapshots/`
4. End the agent session (triggers the `stop` hook)
5. Confirm the git hook was installed: `cat .git/hooks/prepare-commit-msg`
6. Stage and commit the edited file
7. Verify the commit message contains an `AI Attribution Summary` block

To test Claude Code hooks, repeat with `.claude/ai_snapshots/` after a Claude session that writes files.

---

## Configuration in this repo

### `.template-provenance.json`

```json
{
  "schema_version": 1,
  "baseline_commit": "31a36fab633cb138d4ca4c67b68deae7f7b3f336",
  "extra_excludes": [":!**/*.xcconfig.local", ":!ios/.bundle/**"]
}
```

| Field | Value | Notes |
|---|---|---|
| `baseline_commit` | Root commit SHA | Measures drift from the original Expo RN template scaffold |
| `extra_excludes` | RN-specific pathspecs | Excludes local Xcode config and Bundler vendor paths on top of defaults |

To change the baseline (e.g. after a major upstream sync), update `baseline_commit` and document the change. Historical Grafana data will reflect the old baseline for earlier commits.

Verify a new baseline SHA is reachable:

```bash
git cat-file -e <sha>
```

### Default exclusions (automatic)

The metrics script excludes lockfiles, `node_modules/`, `ios/Pods/`, `android/build/`, `.expo/`, coverage output, and other build artifacts without needing to list them in `extra_excludes`. See the upstream [LANGUAGES.md](https://github.com/codeandtheory/ct-github-code-metrics/blob/main/core/docs/LANGUAGES.md) for the full list.

### `.gitignore` entries

```
.claude/ai_snapshots/
.cursor/ai_snapshots/
```

Snapshot directories are never committed.

---

## Required GitHub secrets

Set these in **Settings → Secrets and variables → Actions** (repo-level or org-level):

| Secret | Required | Purpose |
|---|---|---|
| `CT_METRICS_TOKEN` | Yes | GitHub PAT with `repo` read access to `codeandtheory/ct-github-code-metrics` — used to download metric scripts in CI |
| `DEVLAKE_WEBHOOK_URL` | For dashboard | DevLake deployment webhook URL (e.g. `https://devlake.example.com/api/plugins/webhook/connections/1/deployments`) |
| `DEVLAKE_BASIC_AUTH` | For dashboard | Config UI credentials as `user:pass` (default on stock installs: `devlake:merico`) |
| `ANTHROPIC_API_KEY` | For semantic scoring | Anthropic API key with billing enabled — semantic scoring is enabled in this repo's workflow |

Set secrets with the GitHub CLI:

```bash
gh secret set CT_METRICS_TOKEN    --body "<your-pat>"
gh secret set DEVLAKE_WEBHOOK_URL --body "<paste-from-admin>"
gh secret set DEVLAKE_BASIC_AUTH  --body "devlake:your-password"
gh secret set ANTHROPIC_API_KEY   --body "<your-anthropic-key>"
```

The workflow completes and uploads artifacts even when DevLake secrets are missing — it logs `DEVLAKE_WEBHOOK_URL or DEVLAKE_BASIC_AUTH not set — skipping POST.` and exits successfully.

---

## Viewing results in Grafana

After the first successful POST to DevLake:

1. Open the Grafana URL your admin shared
2. Select this repo from the **$repo** dropdown
3. View panels for template %, custom %, AI %, human %, time series, and cross-repo comparison

Deep-link to this repo (replace with your actual repo URL and dashboard UID):

```
https://devlake.example.com/grafana/d/<dashboard-uid>/template-code-metrics?var-repo=https://github.com/<org>/TestProjectReportingKitCursorAgent
```

Admin setup (DevLake, webhook connection, dashboard import) is documented upstream in [SETUP.md](https://github.com/codeandtheory/ct-github-code-metrics/blob/main/core/docs/SETUP.md) and [GRAFANA.md](https://github.com/codeandtheory/ct-github-code-metrics/blob/main/core/docs/GRAFANA.md).

---

## Troubleshooting

### AI attribution block missing from commit messages

| Symptom | Likely cause | Fix |
|---|---|---|
| No block at all | Git hook not installed yet | Run an AI session that writes files, then end the session |
| No block at all | `python3` not on PATH | Install Python 3; hooks fail silently otherwise |
| No block at all | No AI snapshots for staged files | Confirm `.cursor/ai_snapshots/` has JSON files after an AI edit |
| Wrong percentages | Stale snapshot from an old session | Delete contents of `.cursor/ai_snapshots/` and `.claude/ai_snapshots/` |

### CI workflow failures

| Symptom | Likely cause | Fix |
|---|---|---|
| `curl` 401/404 on script download | Missing or invalid `CT_METRICS_TOKEN` | Create a PAT with repo read access to the upstream metrics repo |
| `Cannot determine baseline` | Invalid `baseline_commit` in provenance JSON | Verify SHA with `git cat-file -e <sha>` |
| `current_total_loc: 0` | Over-aggressive exclusions | Review `extra_excludes`; compare with upstream LANGUAGES.md |
| DevLake POST fails | Wrong webhook URL or auth | URL must use port **4000** (Config UI nginx), not 8080 |
| Semantic fields null | `ANTHROPIC_API_KEY` unset or diff too large | Set the secret; diffs over 10k lines skip semantic scoring |

### Grafana shows no data

- Confirm at least one workflow run completed with DevLake secrets set
- Check the workflow log for `Posted to DevLake successfully.`
- The `$repo` dropdown populates from the `repoUrl` in the webhook payload — it may take up to a minute after the first POST

### Updating or reinstalling hooks

To refresh hooks from upstream:

```bash
git clone https://github.com/codeandtheory/ct-github-code-metrics.git /tmp/ct-metrics-setup
cp -r /tmp/ct-metrics-setup/.claude/hooks/. .claude/hooks/
cp -r /tmp/ct-metrics-setup/.cursor/hooks/. .cursor/hooks/
cp /tmp/ct-metrics-setup/.claude/settings.json .claude/settings.json
cp /tmp/ct-metrics-setup/.cursor/hooks.json .cursor/hooks.json
rm -rf /tmp/ct-metrics-setup
```

Re-run an AI session afterward to reinstall the local git hook if needed.

---

## Files added by CT metrics setup

| Path | Purpose |
|---|---|
| `.claude/settings.json` | Claude Code hook configuration |
| `.claude/hooks/*.py` | Claude attribution scripts |
| `.cursor/hooks.json` | Cursor hook configuration |
| `.cursor/hooks/*.py` | Cursor attribution scripts |
| `.template-provenance.json` | Baseline commit and extra exclusions |
| `.github/workflows/template-metrics.yml` | CI pipeline |
| `.gitignore` (updated) | Ignores AI snapshot directories |
