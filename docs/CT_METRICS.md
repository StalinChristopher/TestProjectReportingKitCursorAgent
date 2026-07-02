# CT Metrics — How It Works

**CT Metrics** (Code & Theory metrics) is a two-part measurement system wired into this repo. It answers two questions automatically, without developers doing extra work:

1. **AI vs. Human attribution** — how much of each commit was written by an AI agent (Claude Code / Cursor) vs. edited by a human.
2. **Template vs. Custom code** — how much of the codebase is untouched template scaffolding vs. project-specific custom code.

The results are stamped into commit messages, published as CI artifacts, and pushed to a shared **DevLake → Grafana** dashboard.

---

## Part 1 — AI Attribution (commit-time)

This runs entirely on your machine, driven by editor hooks and a git hook.

### The flow

```
AI edits a file        →  snapshot the exact AI output
   (Claude/Cursor)         (.claude/ai_snapshots or .cursor/ai_snapshots)
        │
   agent turn ends     →  ensure prepare-commit-msg git hook is installed
        │
   git commit          →  diff staged files vs. snapshots
                          →  count AI-authored vs. human-edited lines
                          →  append the attribution summary to the commit message
```

### Components

| Stage | Claude Code | Cursor |
|-------|-------------|--------|
| **Snapshot AI output** (after each file write) | [`snapshot_ai_output.py`](../.claude/hooks/snapshot_ai_output.py) via `PostToolUse` hook in [`settings.json`](../.claude/settings.json) | [`snapshot_ai_output.py`](../.cursor/hooks/snapshot_ai_output.py) via `afterFileEdit` hook in [`hooks.json`](../.cursor/hooks.json) |
| **Install git hook** (when the agent stops) | [`stop_generate_attribution.py`](../.claude/hooks/stop_generate_attribution.py) via `Stop` hook | [`stop_install_git_hook.py`](../.cursor/hooks/stop_install_git_hook.py) via `stop` hook |
| **Compute attribution** (at commit) | [`compute_attribution.py`](../.claude/hooks/compute_attribution.py) | [`compute_attribution.py`](../.cursor/hooks/compute_attribution.py) |

### How attribution is calculated

At commit time, the `prepare-commit-msg` git hook runs `compute_attribution.py`, which for each staged file:

1. Loads the **AI snapshot** — the exact content the agent last wrote (keyed by an MD5 of the absolute path).
2. Runs a line-level diff (`difflib.SequenceMatcher`) between the snapshot and the **currently committed** content.
   - Lines **unchanged** since the AI wrote them → counted as **AI-authored**.
   - Lines **added or replaced** by the human afterward → counted as **human-authored**.
3. Files with **no snapshot** are counted as 100% human.

It then appends a summary to the commit message, e.g.:

```
─────────────────────────────────────────
🤖 AI Attribution Summary (Claude Code)
─────────────────────────────────────────
  Overall  →  AI: 82%  |  Human: 18%
  Per-file breakdown:
    src/screens/HomeScreen.tsx
      [████████░░]  AI 80% (120 lines)  /  Human 20% (30 lines)
─────────────────────────────────────────
```

> The summary is only appended when at least one committed file had an AI snapshot. Merge and squash commits are skipped.

### Coexistence

Both the Claude and Cursor installers write to the **same** `prepare-commit-msg` hook. If a hook already exists, each installer detects its own marker and *prepends/chains* its block rather than overwriting — so a repo using both agents runs both attribution passes.

---

## Part 2 — Template vs. Custom Metrics (CI-time)

This runs in GitHub Actions and measures how much the repo has diverged from its template baseline.

### Trigger

[`.github/workflows/template-metrics.yml`](../.github/workflows/template-metrics.yml) runs on:
- pushes to `main`
- merged pull requests
- manual `workflow_dispatch`

### The baseline

[`.template-provenance.json`](../.template-provenance.json) records the `baseline_commit` (the commit representing the pristine template) and any `extra_excludes`. The template-metrics script diffs the current tree against this baseline to classify code as **template** (unchanged) or **custom** (added/modified).

### Steps

1. **Download scripts** — pulls `template-metrics.mjs` and `ai-attribution-metrics.mjs` from the private `codeandtheory/ct-github-code-metrics` repo (auth: `CT_METRICS_TOKEN`).
2. **Compute template metrics** → `metrics.json` (`template_pct`, `custom_pct`, `total_loc`).
3. **Compute AI attribution metrics** → `ai-attribution.json` (aggregated `ai_pct` / `human_pct` across commits, from the commit-message summaries produced in Part 1).
4. **Upload artifacts** — both JSON files attached to the workflow run.
5. **POST to DevLake** — merges both into `combined-metrics.json` and pushes a deployment-style payload to the DevLake webhook, which surfaces on the shared **Grafana** dashboard.

### Required secrets / variables

| Secret | Purpose |
|--------|---------|
| `CT_METRICS_TOKEN` | Read access to the private metrics-scripts repo |
| `ANTHROPIC_API_KEY` | Used by `template-metrics.mjs` |
| `DEVLAKE_WEBHOOK_URL` | DevLake incoming webhook |
| `DEVLAKE_BASIC_AUTH` | `user:pass` basic-auth for the webhook |

> If `DEVLAKE_WEBHOOK_URL` or `DEVLAKE_BASIC_AUTH` are unset, the workflow computes and uploads metrics but **skips** the POST (no failure).

---

## Setup

CT Metrics was scaffolded by the `setup-ct-metrics` skill, which installs both the AI-attribution hooks and the template-metrics workflow. To reconfigure, re-run that skill or edit the files listed above. The template baseline lives in `.template-provenance.json`; the CI dashboard destination is configured entirely through the GitHub secrets above.
