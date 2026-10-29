# .github

Shared reusable CI/CD workflows for the `tanzhongyan-org` template fleet.
Templates keep thin callers pinned by digest; a change here reaches every
template after its pin update.

## Rules

- Every workflow here is `workflow_call` only. Triggers live in the thin
  template callers.
- Make targets are the command authority: `make check`, `make licenses`,
  `make security`, `make brand-check`, `make audit`, `make dast`, `make e2e`
  and the other check targets. The workflows only set up the toolchain and
  invoke Make.
- Runners are Linux only, with dependency, `.next/cache` and Playwright
  caches (see `setup`).
- Superseded PR runs cancel; scheduled and deploy workflows never cancel.
- Docs-only changes skip `ci` gates but still report `ci-passed`.
- Every action is pinned to a full SHA with a version comment. Renovate
  (`helpers:pinGitHubActionDigests` plus the `# renovate:` annotations)
  raises the updates; a human merges them because shared CI reaches every
  template. There is no automerge in this repo.

## Layout

```text
.github/
  actions/
    setup/    Toolchain plus caches per runtime (internal)
    notify/   notify() composite action
  workflows/
    ci.yml            Gates, licences, security, brand-check, blank-DB migrations
    codeql.yml        CodeQL SAST with security-extended queries
    dast.yml          Schemathesis or ZAP baseline against a running server
    preview-checks.yml E2E smoke, nav crawler, axe, Lighthouse, static routes
    migrate.yml       Pre-deploy migrations over the direct URL
    backup.yml        Weekly pg_dump plus notify() on failure
    eas.yml           EAS build, store submit or PR update
```

Internal `setup`/`notify` references pin the reviewed actions commit
(`actions-sha` comment). Bump them together when the actions change.

## Caller permissions

A reusable workflow cannot escalate beyond what its caller grants, so each
thin caller must set at least these top-level permissions:

| Caller     | Permissions                                                  |
|------------|--------------------------------------------------------------|
| ci         | `contents: read`, `pull-requests: read`                      |
| codeql     | `contents: read`, `security-events: write`, `actions: read`  |
| dast       | `contents: read`                                             |
| preview    | `contents: read`                                             |
| migrate    | `contents: read`                                             |
| backup     | none                                                         |
| eas        | `contents: read`                                             |

`ci-passed` is the single required check for branch rulesets.

## ci.yml

Gates (`make check` on a blank migrated Postgres), `make licenses`,
`make security`, optional `make brand-check`, secret/SCA/workflow scans,
dependency review on PRs, and the final `ci-passed` job.

Call it on push to `main` and on pull requests. The shared Postgres service
uses user `postgres`, password `password`; point `database-url` at it.

| Runtime     | `database-url` shape                                            | `install-command`              | `db-setup-command` |
|-------------|-----------------------------------------------------------------|--------------------------------|--------------------|
| t3          | `postgresql://postgres:password@localhost:5432/<db>`            | `bun install --frozen-lockfile` | `bun run db:push` |
| ts-backend  | `postgresql://postgres:password@localhost:5432/<db>`            | `bun install --frozen-lockfile` | `make db-push`    |
| python      | `postgresql+asyncpg://postgres:password@localhost:5432/<db>`    | `uv sync --locked --all-groups` | `make db-migrate` |
| go          | `postgres://postgres:password@localhost:5432/<db>?sslmode=disable` | `go mod download`            | _(none)_           |
| rust        | `postgres://postgres:password@localhost:5432/<db>`              | _(none)_                       | _(none)_           |

T3 callers pass `brand-check: true` and `playwright: true`.

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
  pull-requests: read
jobs:
  ci:
    uses: tanzhongyan-org/.github/.github/workflows/ci.yml@WORKFLOW_PIN_SHA # v1
    with:
      runtime: t3
      postgres-db: app_db
      database-url: postgresql://postgres:password@localhost:5432/app_db
      install-command: bun install --frozen-lockfile
      db-setup-command: bun run db:push
      brand-check: true
      playwright: true
```

## codeql.yml

CodeQL with `security-extended` queries over the repo language plus the
`actions` language. Call it on push to `main`, on pull requests and weekly.

| Runtime          | `language`              | `build-mode` |
|------------------|-------------------------|--------------|
| t3, ts-backend   | `javascript-typescript` | `none`       |
| python           | `python`                | `none`       |
| go               | `go`                    | `autobuild`  |
| rust             | `rust`                  | `none`       |

```yaml
name: CodeQL
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "17 4 * * 2"
permissions:
  contents: read
  security-events: write
  actions: read
jobs:
  codeql:
    uses: tanzhongyan-org/.github/.github/workflows/codeql.yml@WORKFLOW_PIN_SHA # v1
    with:
      runtime: go
      language: go
      build-mode: autobuild
```

## dast.yml

Weekly DAST, never a PR gate. Schemathesis fuzzes OpenAPI APIs
(`make dast` runs against `BASE_URL`); ZAP runs the baseline scan pinned to
its own container version. The workflow backgrounds `start-command` and polls
`health-url` before scanning.

| Runtime    | `tool`        | `install-command`                  | `db-setup-command` | `build-command` | `start-command` | `health-url` / `base-url` |
|------------|---------------|------------------------------------|--------------------|-----------------|-----------------|---------------------------|
| t3         | `zap`         | `bun install --frozen-lockfile`     | `bun run db:push`  | `bun run build` | `bun run start` | `http://localhost:3000/` |
| ts-backend | `schemathesis`| `bun install --frozen-lockfile`     | `make db-push`     | _(none)_        | `bun src/index.ts` | `http://localhost:3000`, health `/health` |
| python     | `schemathesis`| `uv sync --locked --no-dev`         | `make db-migrate`  | _(none)_        | `uv run uvicorn app.main:app --port 8000` | `http://localhost:8000`, health `/api/v1/health` |
| go         | `zap`         | `go mod download`                   | _(none)_           | `make build`    | `./bin/server`  | `http://localhost:8080`, health `/health` |
| rust       | `zap`         | _(none)_                            | _(none)_           | `cargo build --locked` | `cargo run --locked --quiet` | `http://localhost:8000`, health `/health` |

API templates behind header identity pass the ZAP replacer through
`zap-cmd-options` (see the Go caller below).

```yaml
name: DAST
on:
  schedule:
    - cron: "41 3 * * 2"
  workflow_dispatch:
permissions:
  contents: read
jobs:
  dast:
    uses: tanzhongyan-org/.github/.github/workflows/dast.yml@WORKFLOW_PIN_SHA # v1
    with:
      runtime: go
      tool: zap
      postgres-db: app_db
      database-url: postgres://postgres:password@localhost:5432/app_db?sslmode=disable
      base-url: http://localhost:8080
      health-url: http://localhost:8080/health
      install-command: go mod download
      build-command: make build
      start-command: ./bin/server
      zap-cmd-options: >-
        -z "-config replacer.full_list(0).description=identity
        -config replacer.full_list(0).enabled=true
        -config replacer.full_list(0).matchtype=REQ_HEADER
        -config replacer.full_list(0).matchstring=x-user-id
        -config replacer.full_list(0).replacement=zap-scanner"
```

## preview-checks.yml

Post-deploy checks on `deployment_status` success: E2E smoke, nav crawler,
axe, Lighthouse and the static-route check. The caller maps its own bypass
secret onto `preview-bypass`; targets read `PREVIEW_URL`, `PREVIEW_BYPASS`
and `BYPASS_HEADER_NAME`. Drop reports into `preview-checks/` and they are
uploaded as one artifact.

```yaml
name: Preview checks
on:
  deployment_status:
permissions:
  contents: read
jobs:
  preview:
    if: github.event.deployment_status.state == 'success'
    uses: tanzhongyan-org/.github/.github/workflows/preview-checks.yml@WORKFLOW_PIN_SHA # v1
    with:
      runtime: t3
      preview-url: ${{ github.event.deployment_status.target_url }}
      install-command: bun install --frozen-lockfile
    secrets:
      preview-bypass: ${{ secrets.VERCEL_AUTOMATION_BYPASS }}
```

Narrow the run with `checks`, for example `checks: e2e a11y`, or rename any
target with `e2e-target`, `crawl-target`, `a11y-target`, `lighthouse-target`
or `static-routes-target`.

## migrate.yml

Pre-deploy migrations over the direct URL. The caller runs it on push to
`main` before the deploy step and holds the direct URL as a secret.

```yaml
name: Migrate
on:
  push:
    branches: [main]
permissions:
  contents: read
jobs:
  migrate:
    uses: tanzhongyan-org/.github/.github/workflows/migrate.yml@WORKFLOW_PIN_SHA # v1
    with:
      runtime: t3
      install-command: bun install --frozen-lockfile
      migrate-target: db-migrate
    secrets:
      database-url: ${{ secrets.DIRECT_DATABASE_URL }}
```

## backup.yml

Weekly `pg_dump` in custom format to a private bucket, `notify()` on
failure. Dumps are named `<prefix>-<UTC timestamp>.dump`.

```yaml
name: Backup
on:
  schedule:
    - cron: "0 1 * * 0"
  workflow_dispatch:
permissions: {}
jobs:
  backup:
    uses: tanzhongyan-org/.github/.github/workflows/backup.yml@WORKFLOW_PIN_SHA # v1
    with:
      bucket: s3://my-bucket/db-backups
      aws-region: ap-southeast-1
    secrets:
      database-url: ${{ secrets.DIRECT_DATABASE_URL }}
      aws-access-key-id: ${{ secrets.BACKUP_AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.BACKUP_AWS_SECRET_ACCESS_KEY }}
      notify-webhook: ${{ secrets.BACKUP_NOTIFY_WEBHOOK }}
```

S3-compatible stores pass `s3-endpoint`.

## eas.yml

Mobile releases for callers with `apps/mobile`. Mode `auto` publishes an
EAS update on PRs, builds and submits on tags, and builds otherwise.
Override with `mode: build`, `build-submit` or `update`.

```yaml
name: EAS
on:
  push:
    tags: ["mobile-v*"]
  pull_request:
    paths: ["apps/mobile/**"]
permissions:
  contents: read
jobs:
  eas:
    uses: tanzhongyan-org/.github/.github/workflows/eas.yml@WORKFLOW_PIN_SHA # v1
    with:
      platform: all
    secrets:
      expo-token: ${{ secrets.EXPO_TOKEN }}
```

## notify()

Composite action that posts one message to a Discord, Slack or Telegram
webhook. The payload carries both `text` (Slack, Telegram) and `content`
(Discord); for Telegram use the full `sendMessage` URL with `chat_id` in
the query string.

```yaml
- uses: tanzhongyan-org/.github/.github/actions/notify@WORKFLOW_PIN_SHA # v1
  with:
    webhook-url: ${{ secrets.NOTIFY_WEBHOOK }}
    message: "Deploy of ${{ github.ref_name }} failed"
```

## License

MIT, holder Tan Zhong Yan. See [LICENSE](LICENSE).
