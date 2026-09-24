# workflows-platform

> Reusable workflows for the platform team

## Which major version to pin

- **`@v1`** — for apps still on legacy `yarn 1` and `@dhis2/cli-style`.
- **`@v2`** — for apps that have migrated to `pnpm` and dropped `@dhis2/cli-style`
  in favour of the shared `@dhis2/config-*` packages.
- **`@pnpm`** (branch, not a tag) — for apps that have moved to `pnpm` but are
  not yet ready to drop `@dhis2/cli-style`. See
  [`dhis2/aggregate-data-entry-app#481`](https://github.com/dhis2/aggregate-data-entry-app/pull/481)
  for an example migration.

Consuming apps must pick the tag that matches their own tooling; the two
major versions are not interchangeable. Apps on `@v2` use `pnpm` (via
`pnpm/action-setup@v4`, version resolved from each workflow's own
`package.json` `packageManager` field) and Node 24, unless noted otherwise.

## Reusable workflows

These live under `.github/workflows/` and are consumed by other repos with
`uses: dhis2/workflows-platform/.github/workflows/<file>.yml@<tag>`.

| Workflow | Purpose | Key inputs | Required secrets |
| --- | --- | --- | --- |
| `lint.yml` | Runs `pnpm d2-app-scripts i18n generate` then `pnpm lint` on the consuming app. | – | – |
| `lint-commits.yml` | Lints the pushed or pull request commits with `commitlint` using the app's commitlint config if it has one, otherwise [`@dhis2/config-commitlint`](https://www.npmjs.com/package/@dhis2/config-commitlint). | – | – |
| `lint-pr-title.yml` | Lints the pull request title with `commitlint` (same config lookup as `lint-commits.yml`), so it can drive semantic-release/squash-merge commit messages. | – | – |
| `test.yml` | Runs `pnpm d2-app-scripts i18n generate` then `pnpm d2-app-scripts test`. | – | – |
| `e2e.yml` | Runs Cypress e2e tests against a local DHIS2 backend cluster (`@dhis2/cli-cluster`) and the app dev server. | – | `CYPRESS_LOGIN_NAME`, `CYPRESS_LOGIN_PASSWORD` |
| `legacy-e2e.yml` | Runs Cypress e2e tests against the debug.dhis2.org instance instead of a local cluster. | `api_version` (default `41`) | – |
| `deploy-pr.yml` | Builds the app and deploys a PR preview to Netlify, with PR/commit comments and status checks enabled. | – | `DHIS2_BOT_GITHUB_TOKEN`, `DHIS2_BOT_NETLIFY_TOKEN`, `NETLIFY_SITE_ID` |
| `deploy-branch.yml` | Builds the app and deploys a non-production branch preview to Netlify, aliased to the branch name. | `branch` (required) | `DHIS2_BOT_GITHUB_TOKEN`, `DHIS2_BOT_NETLIFY_TOKEN`, `NETLIFY_SITE_ID` |
| `deploy-production.yml` | Builds the app and deploys it as the production site on Netlify. | `branch` (default `main`) | `DHIS2_BOT_GITHUB_TOKEN`, `DHIS2_BOT_NETLIFY_TOKEN`, `NETLIFY_SITE_ID` |
| `release.yml` | Builds the app, signs commits, runs `dhis2/action-semantic-release` to cut a release (apphub and/or GitHub), then uploads the build via `dhis2/deploy-build`. | `publish_apphub` (default `true`), `apphub_channel` (default `stable`), `publish_github` (default `true`), `skip_deploy_build` (default `false`) | `DHIS2_BOT_GITHUB_TOKEN`, `DHIS2_BOT_APPHUB_TOKEN` |
| `generate-and-upload-bom.yml` | Generates a CycloneDX SBOM (`cdxgen`) for the app and uploads it to Dependency Track. | `node_version` (default `22`), `project_id` (required), `dependency_track_url` (default `https://dt.security.dhis2.org/api/v1/bom`) | `DEPENDENCYTRACK_APIKEY` |
| `comment-and-close.yml` | Closes a given issue with a comment pointing reporters to the DHIS2 issue tracker (used for triaging issues opened in app repos). | `issue_number` (required) | – |

## Usage examples

Pin to a released major tag (e.g. `@v1`) rather than `@main`, so updates to
this repo don't silently change behaviour in consuming apps. `secrets: inherit`
passes through all secrets available to the caller workflow, which is how
most app repos wire up the required secrets below.

### `@v2` (pnpm + shared configs, no `@dhis2/cli-style`)

Same shape as the `@pnpm` migration in
[`dhis2/aggregate-data-entry-app#481`](https://github.com/dhis2/aggregate-data-entry-app/pull/481),
pinned one step further once `@dhis2/cli-style` is also dropped:

```yaml
name: test-and-release

on: push

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ !contains(fromJSON('["refs/heads/master", "refs/heads/main"]'), github.ref) }}

jobs:
    lint-commits:
        uses: dhis2/workflows-platform/.github/workflows/lint-commits.yml@v2
    lint:
        uses: dhis2/workflows-platform/.github/workflows/lint.yml@v2
    test:
        uses: dhis2/workflows-platform/.github/workflows/test.yml@v2
    e2e:
        uses: dhis2/workflows-platform/.github/workflows/legacy-e2e.yml@v2
        if: '!github.event.push.repository.fork'
        secrets: inherit
        with:
            api_version: 43
    release:
        needs: [lint-commits, lint, test, e2e]
        uses: dhis2/workflows-platform/.github/workflows/release.yml@v2
        if: '!github.event.push.repository.fork'
        secrets: inherit
```

> [!NOTE]
> The examples below pin `@v1`. For an app already on `pnpm` with `@dhis2/cli-style`
> dropped, swapping `@v1` for `@v2` in each `uses:` line should be enough. No other changes needed.

### `lint.yml`, `lint-commits.yml`, `test.yml`, `legacy-e2e.yml`, `release.yml` (`@v1`)

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/test-and-release.yml),
one workflow file fanning out to five reusable workflows on every push, with
`release` gated on the others passing:

```yaml
name: test-and-release

on: push

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ !contains(fromJSON('["refs/heads/master", "refs/heads/main"]'), github.ref) }}

jobs:
    lint-commits:
        uses: dhis2/workflows-platform/.github/workflows/lint-commits.yml@v1
    lint:
        uses: dhis2/workflows-platform/.github/workflows/lint.yml@v1
    test:
        uses: dhis2/workflows-platform/.github/workflows/test.yml@v1
    e2e:
        uses: dhis2/workflows-platform/.github/workflows/legacy-e2e.yml@v1
        if: '!github.event.push.repository.fork'
        secrets: inherit
        with:
            api_version: 43
    release:
        needs: [lint-commits, lint, test, e2e]
        uses: dhis2/workflows-platform/.github/workflows/release.yml@v1
        if: '!github.event.push.repository.fork'
        secrets: inherit
```

### `lint-pr-title.yml`

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/lint-pr-title.yml):

```yaml
name: lint-pr-title

on:
    pull_request:
        types: ['opened', 'edited', 'reopened', 'synchronize']

concurrency:
    group: ${{ github.workflow }}-${{ github.head_ref }}
    cancel-in-progress: true

jobs:
    lint-pr-title:
        uses: dhis2/workflows-platform/.github/workflows/lint-pr-title.yml@v1
```

### `e2e.yml`

From [`dhis2/scheduler-app`](https://github.com/dhis2/scheduler-app/blob/master/.github/workflows/test-and-release.yml):

```yaml
e2e:
    uses: dhis2/workflows-platform/.github/workflows/e2e.yml@v1
    if: '!github.event.push.repository.fork'
    secrets: inherit
```

### `deploy-pr.yml`

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/deploy-pr.yml):

```yaml
name: deploy-pr

on:
    pull_request:

concurrency:
    group: ${{ github.workflow }}-${{ github.head_ref }}
    cancel-in-progress: true

jobs:
    deploy:
        uses: dhis2/workflows-platform/.github/workflows/deploy-pr.yml@v1
        if: '!github.event.pull_request.head.repo.fork'
        secrets: inherit
```

### `deploy-branch.yml`

From [`dhis2/aggregate-data-entry-app`](https://github.com/dhis2/aggregate-data-entry-app/blob/master/.github/workflows/deploy-branch.yml):

```yaml
name: deploy-branch

on:
    push:
        branches:
            - development

concurrency:
    group: ${{ github.workflow }}-${{ github.head_ref }}
    cancel-in-progress: true

jobs:
    deploy:
        uses: dhis2/workflows-platform/.github/workflows/deploy-branch.yml@v1
        secrets: inherit
        with:
            branch: development
```

### `deploy-production.yml`

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/deploy-production.yml):

```yaml
name: deploy-production

on:
    push:
        branches:
            - master

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true

jobs:
    deploy:
        uses: dhis2/workflows-platform/.github/workflows/deploy-production.yml@v1
        secrets: inherit
        with:
            branch: master
```

### `generate-and-upload-bom.yml`

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/generate-and-upload-bom.yml),
run nightly rather than on every push:

```yaml
name: 'This workflow creates bill of material and uploads it to Dependency-Track each night'

on:
    schedule:
        - cron: '0 0 * * *'

concurrency:
    group: ${{ github.workflow }}-${{ github.head_ref }}
    cancel-in-progress: true

jobs:
    create-bom:
        uses: dhis2/workflows-platform/.github/workflows/generate-and-upload-bom.yml@v1
        with:
            node_version: 20
            project_id: '08257ca6-e0dc-4bd8-a3b0-f3608f406cf8'
        secrets: inherit
```

### `comment-and-close.yml`

From [`dhis2/user-app`](https://github.com/dhis2/user-app/blob/master/.github/workflows/comment-and-close.yml):

```yaml
name: comment-and-close

on:
    issues:
        types: [opened]

jobs:
    comment-and-close:
        uses: dhis2/workflows-platform/.github/workflows/comment-and-close.yml@v1
        if: '!contains(fromJson(''["dhis2-bot", "kodiakhq", "dependabot"]''), github.event.issue.sender.login)'
        with:
            issue_number: ${{ github.event.issue.number }}
```

## This repo's own automation

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `release-workflows.yml` | Push to `main` | Runs `semantic-release` for this repo itself, so consuming apps can pin a released major tag (e.g. `@v1`) of these reusable workflows. |
