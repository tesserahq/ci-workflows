# Dependabot automation for portals

Status: pilot in progress
Owner: platform / portals
Related: `tesserahq/ci-workflows` (shared workflows, `v0.1.0`), `tesserahq/togly-portal` PR #39 (pilot), `tesserahq/togly-portal` PR #37 (auto-merge disabled), PR #5 in `tesserahq/.github` (closed)

## Problem

Portals receive Dependabot updates as separate pull requests. Major updates often fail lint, typecheck, tests, or the build, and each failure is diagnosed by hand. The backlog grows, and portals fall behind on dependencies.

We need one process that:

- gives every portal the same validation, including a production build;
- groups related updates so there are fewer, more coherent PRs;
- auto-merges routine updates only after the relevant checks pass on the exact commit;
- leaves failing updates and anything outside policy for a human.

## Goals and non-goals

Goals:

- One validation contract shared by all portals.
- Policy for auto-merge defined in one reviewable place.
- Merges bound to the commit that was checked.
- Failing or out-of-policy updates are visible, not silently stuck.

Non-goals:

- Auto-merging production dependency majors. These stay human-reviewed, one PR each.
- Auto-merging GitHub Actions updates. These stay human-reviewed.
- Replacing Dependabot's own version detection.
- FastAPI services. They have the same backlog, but they are out of scope for this iteration. They will adopt the same validation contract and policy file in a follow-up, so the design is not shaped around Bun alone.

## Current state

- The shared test workflow (`portal-tests.yml`) runs the build when a `build` script exists. This was added in `tesserahq/.github` #6, and `ci-workflows` includes it as of `5c53674`. The build is therefore already gated inside the `tests` check. A separate `build` job is optional, not required.
- The shared auto-merge gate auto-merges patch and minor updates after `code-quality`, `typecheck`, and `tests / tests` pass. Major tooling eligibility from PR #5 is not merged. PR #5 is closed.
- **Observed on togly-portal PR #37:** auto-merge was enabled by an older gate pin (`ba740570`) that has no wait step. The gate went straight from metadata to `gh pr merge --auto`, while `typecheck` was failing. Auto-merge is now disabled on #37. The gate also ignored `maintainer-changes: true` in that group. See Policy below.
- Between the head-SHA check and `gh pr merge --auto`, a new commit can land. Auto-merge is then enabled on a commit that was never checked. Branch protection prevents the merge on repos that require checks. The race matters on repos without it.
- `portal-tests.yml` reports success when a portal has no test script or no test files, because its steps are skipped with `if:` and the job still concludes `success`. A portal with no tests therefore passes the gate trivially. Open question 1 is about this behavior.

## Design

The design has four parts. Each lives in one place.

### 1. Validation contract: one reusable workflow, every portal calls it

A reusable workflow in `tesserahq/ci-workflows` runs the same jobs for every portal:

| Job | Purpose | Gates auto-merge |
|-----|---------|------------------|
| `lint` | Style and static rules | Yes |
| `typecheck` | Type errors | Yes |
| `tests` | Unit and integration tests, plus the build when a build script exists | Yes |

Rules:

- **Check names.** A reusable workflow's check run name is `<caller job id> / <called job name>`. The target is for the caller template to fix the job id as `ci`, so the checks become `ci / lint`, `ci / typecheck`, `ci / tests`. The pilot keeps the names portals already use (`tests / tests`, `typecheck`, `code-quality`), so branch protection needs no changes yet. Migrating to `ci / *` is a later step that needs matching branch protection changes in each repo.
- **A missing check counts as failing, never as skipped.** Branch protection treats a skipped check as passing. So the job must always run and conclude `failure` when a portal has no test script or no test files. The portal must opt out explicitly through its policy file, not through a silent `if:` skip. This means `portal-tests.yml` needs a change: its current `if:` steps let the job conclude `success` with no tests.
- **Branch protection is the primary control.** Each portal's `main` branch protection requires the validation checks. The org is on the free plan, so org rulesets are unavailable. Per-repo branch protection works because all portals are public. Each repo has to be configured individually. The auto-merge gate checks the same names, so it still works if a repo's protection is missing.

### 2. Grouping: a shared convention in each portal's `dependabot.yml`

Every portal uses the same group names:

- `dev-tooling`: build, bundler, CSS, lint, test, and type packages. Dev dependencies only.
- `prod-minor-patch`: production minor and patch updates.
- `actions`: GitHub Actions updates. Never auto-eligible.

Production majors are **not grouped**. Each one gets its own PR, because each usually needs its own migration. Groups are split by scope so one change is reviewed as one thing.

togly-portal already defines `bun-major-build-tooling`, `bun-major-css-tooling`, `bun-major-lint-tooling`, `bun-major-test-tooling`, `bun-major-types-tooling`, and `bun-patch-and-minor`. The policy maps those names to the shared concepts above. Portals keep their own group names. The shared workflow reads the group name from the policy, and it never hard-codes a list of groups that could drift.

### 3. Policy as data, evaluated per dependency

A single policy file defines which updates are eligible. Portals use the shared default, and may override it in `.github/dependabot-policy.yml`.

Rules are evaluated top to bottom, first match wins. Deny rules come first, and the file ends with an explicit default deny, so an update type or ecosystem we did not anticipate fails closed:

```yaml
# Shared default. Evaluated per dependency entry; first match wins.
# Anything unmatched is not eligible.
rules:
  - maintainer-changes: true      # Dependabot flagged a maintainer change: human review
    eligible: false
  - ecosystem: github_actions
    eligible: false
  - update-type: major
    dependency-type: direct:development
    group: dev-tooling
    eligible: true
  - update-type: major
    eligible: false               # every other major: human review
  - update-type: [patch, minor]
    eligible: true
  - eligible: false               # default deny
```

**Evaluation is per dependency, not per PR.** The workflow reads `updated-dependencies-json` from `dependabot/fetch-metadata` and checks every entry against the rules. The PR is `eligible` only if every entry matches an eligible rule.

The shape of each entry, from the `fetch-metadata` source at `v3.1.0` (`25dd0e3`), is an object with camelCase keys: `dependencyName`, `dependencyType`, `updateType`, `directory`, `packageEcosystem`, `targetBranch`, `prevVersion`, `newVersion`, `compatScore`, `maintainerChanges`, `dependencyGroup`, `alertState`, `ghsaId`, `cvss`. The output is a JSON array. The workflow must parse it before use, for example with `fromJSON()` or `jq`. This was read from source. It still needs confirmation on a live grouped PR.

The group-level outputs are summaries, and they lose information:

- `update-type` is the highest semver change in the PR, so one major makes it major.
- `dependency-type` is the highest-priority type, so one `direct:production` entry makes the PR `direct:production`. This is fail-safe for the current gate, but it hides which entries are development-only.
- `dependency-group` comes from the first entry only, so the group name can be wrong for a mixed group.

Per-entry evaluation avoids all three problems.

The shared workflow writes a single `eligible` output. Both the wait step and the merge step read that output. The duplicated `if:` in PR #5 goes away. Changing policy is a reviewed change to a data file.

### 4. Merge: only after checks pass on the exact commit

1. Wait for the required checks on `HEAD_SHA`.
2. Verify the PR head still equals `HEAD_SHA`.
3. Enable auto-merge bound to that SHA, for example `gh pr merge --auto --squash --match-head-commit "$HEAD_SHA" "$PR_URL"`. Confirm `gh` enforces the match when `--auto` is set.

Where branch protection requires checks, GitHub re-gates auto-merge on any new push, so the race above only applies to repos without it. The SHA pin is kept as defense in depth. It does not replace branch protection.

Supply chain, for majors that are eligible under policy:

- Bun already skips dependency lifecycle scripts unless a package is listed in `trustedDependencies`. Review changes to `trustedDependencies` in the PR as an explicit signal. Do not add `--ignore-scripts`, because it can break trusted packages that need their postinstall step.
- Tests and the build still execute new code during validation. The gate therefore relies on cooldown, the lockfile diff, and the limit to dev tooling, not on script suppression.
- Enable a Dependabot `cooldown` so a release has time to be reported before it is picked up.
- Keep the lockfile diff visible in the PR so reviewers can see transitive changes.

## Alternatives considered

**Renovate.** Renovate supports shared presets in `.github`, per-rule automerge, `minimumReleaseAge`, and grouping natively. It would replace most of the custom workflow in this design.

We recommend staying on Dependabot for now, for these reasons:

- Dependabot is already enabled across the portals, and its security updates are already in use.
- Adopting Renovate means a new app to install and approve, and a migration of every portal's config.
- The custom workflow in this design is small once the policy is data. It stays a fallback for the Dependabot path only.

Revisit this decision if the policy rules outgrow the workflow, or if the merge gate becomes harder to maintain than a Renovate preset.

## Failure and backlog handling

- A failing update is not left silent. The workflow adds the `needs-human` label and comments the failing job name and a link to its log.
- An update that is out of policy gets the same label, with a reason ("production major", "GitHub Actions", "maintainer changes").
- Production majors stay as individual PRs. They are not grouped.
- A weekly report (issue or summary) lists open `needs-human` PRs per portal, so the backlog is visible.

## Rollout

1. **Pilot on togly-portal.** In progress. `tests.yml`, `dependabot-auto-merge.yml`, and the formatting check are on `chore/use-ci-workflows`, PR #39, pinned to `ci-workflows` `v0.1.0` (`5c53674`). The first run passed `code-quality`, `typecheck`, and `tests / tests`. Repository formatting already passes `format:check`, so no reformatting is needed. Still to confirm: the gate's behavior on a real grouped Dependabot PR, including the `updated-dependencies-json` shape, the `maintainer-changes` rule, and the no-auto-merge-on-failure behavior seen on #37.
2. **Fix the gaps before widening.** Make missing checks fail, not pass (open question 1). Add branch protection requiring the validation checks on each portal's `main`.
3. **Roll out to the other portals.** Each portal gets the same caller workflow, group config, and policy reference. No per-portal logic.
4. **Review after one cycle.** Count auto-merged updates, reverts, and `needs-human` items. Adjust policy based on those numbers.

## Risks

- **A passing check does not prove safety.** Checks run the code, so a compromised release can pass them. Mitigated by keeping production majors manual and individual, Bun's default script blocking, cooldown, eligibility limited to dev tooling, and the `maintainer-changes` rule.
- **Gaps in test coverage.** A portal with thin tests can auto-merge a broken update. Mitigated by the build inside the tests check and by counting no-tests as failing (open question 1).
- **Stale gate pins.** A Dependabot PR created before a gate change keeps its old pin, as PR #37 showed. Mitigated by making the gate's own behavior correct, not by relying on PR branches being current.
- **Policy drift between portals.** Mitigated by the shared default and a single policy evaluation.
- **Check names change.** If a caller renames its job, gates can wait for names that never appear. Mitigated by the caller template fixing the job id, and by a timeout that fails closed.

## Open questions

Resolved:

- **No-tests behavior (decided).** A portal with no test script or files fails the gate and must opt out explicitly through its policy file. `dev-tooling` gets no exemption.
- **Policy ownership (decided).** The repo owner maintains the shared default policy. The team of three portal developers changes it through reviewed PRs.
- **Dependabot vs. Renovate (decided).** Stay on Dependabot. See Alternatives considered.
- **Required checks (decided).** Per-repo branch protection, not org rulesets. The org is on the free plan, where org rulesets are unavailable. The portals are public, so branch protection is available on each one.
- **Check names for the pilot (decided).** Keep the names portals already use. Migration to `ci / *` is a later step.

Open:

1. **`updated-dependencies-json` on a live PR.** The shape above comes from `fetch-metadata` source at `v3.1.0`. It needs confirmation on a real grouped Dependabot PR during the pilot.

## Decision on PR #5

Closed. It hard-coded group names, duplicated the eligibility condition, had the merge race, and had no build gate. Its core idea, allowing major updates to dev tooling when checks pass, survives as one rule in the policy file.
