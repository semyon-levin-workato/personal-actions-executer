# Coding Agent Instructions

## What this is

Personal GitHub Actions automation. There is no application code, build, or test suite. Everything lives in `.github/workflows/`. The origin is `semyon-levin-workato/personal-actions-executer`, a downstream copy of the upstream template `remal/personal-actions-executer`.

## Upstream sync (read before editing any workflow)

- `.github/workflows/manage-notifications.yml` is synced from `remal/personal-actions-executer` on a regular basis (hourly) by `sync-actions.yml`. The `FILES` env lists which paths are synced. The step downloads the upstream copy and pushes it back, so **any local edit to `manage-notifications.yml` is overwritten on the next sync run.**
- **When you change `manage-notifications.yml`, change it in two places: this repo and `remal/personal-actions-executer`.** `remal/personal-actions-executer` takes priority: if the two copies differ, take the `remal` version as the base.
- This repo and `remal` are managed by two different GitHub accounts. The `~/.bin/gh` wrapper selects the account by working directory, so make the `remal` changes from the `remal` checkout and changes to this repo from here.
- `sync-actions.yml` and `bump-repository-activity.yml` are local to this repo and safe to edit here.

## Workflows

Each workflow also runs on `workflow_dispatch` and on push to `main` that touches its own file.

- **manage-notifications.yml** (cron every 15 min): the main logic. Uses `secrets.NOTIFICATIONS_TOKEN` to fetch the authenticated user's notifications from the last 3 days and mark as done the ones that do not need attention: cancelled check-suite runs, duplicate check-suites (keeps only the newest per repo + workflow + branch), `review_requested` notifications where the user is not actually a requested reviewer, and merged or closed PRs that are bot PRs, carry the `sync-with-template` label, or whose repo is listed in the dismiss variables. State lives in a two-file cache (`.notifications-cache/processed.txt` and `done.txt`) persisted with `actions/cache`. The cache key includes the workflow file SHA and a hash of the dismiss settings, so editing either invalidates the cache.
- **sync-actions.yml** (cron hourly): the workflow that syncs `manage-notifications.yml`. It downloads the paths listed in its `FILES` env from `remal/personal-actions-executer` and pushes back any changes. See Upstream sync above.
- **bump-repository-activity.yml** (cron daily): writes a timestamp to `repository-activity.bumper` and commits it via `remal-github-actions/bump-repository-activity`, keeping the repo active so GitHub does not disable the scheduled workflows.

## Configuration

- Secrets: `PUSH_BACK_TOKEN` (used for the commits pushed back by sync and bump; the bump falls back to `github.token`), `NOTIFICATIONS_TOKEN` (a PAT with notifications access for the user whose notifications are managed).
- Repo variables, each a newline-separated list of `owner` or `owner/repo` entries:
  - `DISMISS_FAILED_PIPELINE_REPOS`: also mark done check-suite notifications from these.
  - `DISMISS_CLOSED_PR_NOTIFICATION_REPOS`: also dismiss merged or closed PR notifications from these.

## Conventions

- **Rate-limit guard.** API-heavy workflows first read core rate-limit usage via `remal-github-actions/get-rate-limits` and skip on scheduled runs when usage is high (at or above 50 for notifications, at or above 75 for the bump). Manual and other non-schedule runs proceed anyway.
- **Push-back.** Workflows that change files commit them back with a `[push-back]` message prefix.

## Commands

- Validate workflows: `actionlint` (run from the repo root).
- Trigger manually: `gh workflow run <workflow-file>.yml`.
- Inspect runs: `gh run list`, `gh run watch`.
