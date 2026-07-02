# Contributing

1. Branch off `main`: `git checkout -b feat/your-change`.
2. Keep code in `src/`, organized by feature folder.
3. Before committing, run: `npm run lint && npm run typecheck && npm run test:ci`.
4. Commits are auto-stamped with AI attribution — see [docs/CT_METRICS.md](docs/CT_METRICS.md).
5. Open a PR into `main`; merged PRs trigger the metrics workflow.
6. Never commit secrets or `.env.*` files.

Questions? See [docs/PROJECT_OVERVIEW.md](docs/PROJECT_OVERVIEW.md).
