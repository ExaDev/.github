# .github

Org-wide GitHub configuration and reusable workflows for [ExaDev](https://github.com/ExaDev). `profile/README.md` is the public org profile shown on [github.com/ExaDev](https://github.com/ExaDev); this file documents the repo's other contents.

## Reusable workflows

### `sibling-dependency-update.yml`

Called via `workflow_call` from a consumer repo's own thin stub. Handles keeping an ExaDev repo's dependencies on other ExaDev-published npm packages up to date automatically:

- On a `repository_dispatch` event of type `sibling-released` (sent by a sibling package's own CI the moment it publishes), bumps the named dependency, opens a PR authenticated as a GitHub App installation (so the PR's own commit triggers the caller repo's CI), and rebase-merges it once every check-run on the PR's head commit reports success.
- On `push` to `main` or a manual `workflow_dispatch`, regenerates any open `sibling-update/*` PR that fell behind `main` and can no longer merge cleanly.

A consumer repo opts in with:

```yaml
name: Sibling dependency instant update
on:
  repository_dispatch:
    types: [sibling-released]
  push:
    branches: [main]
  workflow_dispatch: {}
jobs:
  sync:
    uses: ExaDev/.github/.github/workflows/sibling-dependency-update.yml@main
    secrets: inherit
```

Requires the calling repo to have access to the `AUTOMERGE_APP_PRIVATE_KEY` secret (an org-level secret, scoped per-repo via its selected-repositories list) and a `pnpm-workspace.yaml` `minimumReleaseAgeExclude` entry for every sibling package it depends on, so `pnpm add` doesn't get blocked by the org's supply-chain minimum-release-age gate on a package that was just published by another ExaDev repo.

### `sync-ecosystem.yml`

Runs directly in this repo, `workflow_dispatch`-triggered. Answers "sync every documents.js-family repo's dependencies right now" in one action instead of waiting for each package's own publish event, or dispatching to repos one at a time by hand: reads every consumer repo's `package.json`, compares each declared sibling-package dependency against that package's current npm `latest` version, and fires the same `sibling-released` dispatch a fresh publish would send for anything found stale. The receiving repo's own `sibling-dependency-update.yml` stub handles it exactly as it would a real publish.

## Branch protection

Every documents.js-family repo's `main` branch requires its own CI's PR-triggered checks (commit linting, lint, typecheck, test, and format-specific smoke/build/manifest checks — never the push-only release/publish/deploy jobs, which never run on a PR) to pass before merging, and requires a PR to be up to date with `main` before merging. This is a platform-level backstop: even a bug in the automation above cannot merge a PR past a pending or failing required check.
