# Project Overview

**TestProjectReportingKitCursorAgent** is an **Expo / React Native** application built on the Code & Theory React Native template. It doubles as a testbed for two internal tooling kits: the **Project Reporting Kit** and **CT Metrics**.

## What the app is

A cross-platform (iOS / Android / web) Expo SDK 55 app using the standard C&T template stack:

| Concern | Choice |
|---------|--------|
| Framework | Expo SDK ~55, React Native 0.83, React 19 |
| Language | TypeScript |
| Navigation | React Navigation (native-stack, bottom-tabs, drawer) |
| Data fetching | TanStack Query + Axios |
| Storage | react-native-mmkv |
| i18n | i18next / react-i18next |
| Lists / UI | FlashList, Reanimated, Gorhom Bottom Sheet |
| Theming | Design-token driven design system |
| Env flavors | `development` / `staging` / `production` via `APP_ENV` + `app.config.ts` |

Source lives under [`src/`](../src/) — organized by feature area (`navigation`, `screens`, `components`, `designSystem`, `theme`, `api`, `services`, `query`, `settings`, `connectivity`, `config`, `utils`). Design tokens are generated from [`tokens/`](../tokens/) via `npm run generate:tokens`.

### Running locally

```sh
npm install
npm start            # development flavor
npm run ios          # prebuild + run iOS (dev)
npm run android      # prebuild + run Android (dev)
```

Flavored variants: `npm run start:qa`, `start:prod`, `ios:qa`, `android:prod`, etc. See [`package.json`](../package.json) for the full script list.

### CI / build notes

The repo integrates with the shared `rn.yml` pipeline from `template-pipeline-react-native`. Workflow integration requires committing the prebuilt `android/` and `ios/` native directories, root `Gemfile`/`Gemfile.lock`, and running the bootstrap script for Fastlane IDs. See the [main README](../README.md) for the detailed CI, prebuild, and CocoaPods guidance.

**JS quality gates:** `npm run lint`, `npm run format:check`, `npm run typecheck`, `npm run test:ci`.

## The two integrated kits

This repo isn't just an app — it's where two C&T automation kits are exercised end-to-end.

### 1. Project Reporting Kit

Located in [`project-reporting/`](../project-reporting/) with a scheduled workflow at [`.github/workflows/daily_project_report.yml`](../.github/workflows/daily_project_report.yml). It generates automated daily project reports by pulling from Jira and posting to Slack. Configuration lives in [`reporting.config.json`](../project-reporting/reporting.config.json):

- **Jira** project `ROC` on `codeandtheory.atlassian.net`
- **Slack** channel `#reporting-work-temp`
- Reports run on a weekday cron (`30 3 * * 1-5` UTC), covering a 09:00–18:00 `Asia/Kolkata` workday window

Secrets and setup are documented in [`REPORTING_SECRETS_CHECKLIST.md`](../project-reporting/REPORTING_SECRETS_CHECKLIST.md) and the [`docs/`](../project-reporting/docs/) folder (Atlassian automation, GitHub secrets, Rovo output contract).

### 2. CT Metrics

Automated measurement of **AI-vs-human** authorship and **template-vs-custom** code, stamped into commits and pushed to a shared Grafana dashboard. See **[CT_METRICS.md](./CT_METRICS.md)** for the full explanation.

## Repository map

```
├── App.tsx, index.ts, app.config.ts   App entry + Expo config (flavors)
├── src/                               Application source (feature folders)
├── tokens/                            Design tokens → generated design system
├── __tests__/                         Jest tests
├── project-reporting/                 Project Reporting Kit (Jira → Slack)
├── .github/workflows/
│   ├── template-metrics.yml           CT Metrics CI (template + AI %)
│   └── daily_project_report.yml       Scheduled project report
├── .claude/ & .cursor/                Agent hooks, rules, skills, AI snapshots
├── .template-provenance.json          Template baseline for metrics
└── docs/                              This documentation
```
