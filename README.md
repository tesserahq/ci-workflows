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

Private repositories need this repo's Actions access setting to allow callers from the organization.

## Status

Initial import. The workflows here are copies of the originals in `tesserahq/.github`, which stay in place until portal callers are switched over.
