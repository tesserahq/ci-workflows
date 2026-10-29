# ci-workflows

Shared reusable GitHub Actions workflows for tesserahq portals.

## Workflows

| File | Purpose |
|------|---------|
| `.github/workflows/portal-tests.yml` | Runs portal tests when a test script and test files exist. |
| `.github/workflows/dependabot-auto-merge.yml` | Gates Dependabot PRs on portal checks before enabling auto-merge. |

## Design

See [`docs/design/dependabot-automation.md`](docs/design/dependabot-automation.md) for the planned validation contract, grouping, policy, and merge rules.

## Usage

Portals call these workflows at a pinned tag or commit SHA, not `main`, so a change here does not silently affect every portal.

```yaml
jobs:
  tests:
    uses: tesserahq/ci-workflows/.github/workflows/portal-tests.yml@<tag-or-sha>
    with:
      bun-version: "1.x"
```

This repo is public, so any portal can call it.

### Dependabot auto-merge

The caller must grant `checks: read`, because the gate reads check runs. Pass the check names that must pass as `required-checks`, one per line. The default is the TypeScript set (`code-quality`, `typecheck`, `tests / tests`). An empty list fails closed.

```yaml
jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    permissions:
      checks: read
      contents: write
      pull-requests: write
    uses: tesserahq/ci-workflows/.github/workflows/dependabot-auto-merge.yml@<tag-or-sha>
    with:
      required-checks: |
        build
```

Known gap: `gh pr merge --auto` is not bound to the checked commit. A push between the check and the merge can still be merged. Adding `--match-head-commit` should close it, but this has not been verified against `gh`'s `--auto` behavior.

## Status

Initial import. The workflows here are copies of the originals in `tesserahq/.github`, which stay in place until portal callers are switched over.
