# Dependabot automation for portals

Status: draft for team review
Owner: platform / portals
Related: PR #5 (`chore/dependabot-major-tooling-gates`), `.github/workflows/dependabot-auto-merge.yml`, `.github/workflows/portal-tests.yml`

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

## Current state (what PR #5 would change)

- `dependabot-auto-merge.yml` auto-merges patch and minor updates after `code-quality`, `typecheck`, and `tests / tests` pass.
- PR #5 extends that to majors in five hard-coded `bun-major-*-tooling` groups. The eligibility condition is duplicated in two steps.
- The gate does not include a build check.
- Between the head-SHA check and `gh pr merge --auto`, a new commit can land. Auto-merge is then enabled on a commit that was never checked.
- `portal-tests.yml` reports success when a portal has no test script or no test files, because its steps are skipped with `if:` and the job still concludes `success`. A portal with no tests therefore passes the gate trivially. See open question 1.

## Design

The design has four parts. Each lives in one place.

### 1. Validation contract: one reusable workflow, every portal calls it

A reusable workflow in `tesserahq/.github` runs the same jobs for every portal:

| Job | Purpose | Gates auto-merge |
|-----|---------|------------------|
| `lint` | Style and static rules | Yes |
| `typecheck` | Type errors | Yes |
| `tests` | Unit and integration tests | Yes |
| `build` | Production build succeeds | Yes (new) |

Rules:

- **Check names are fixed by the caller template.** A reusable workflow's check run name is `<caller job id> / <called job name>`. The caller template fixes the job id as `ci`, so the checks are always `ci / lint`, `ci / typecheck`, `ci / tests`, and `ci / build`. The gate and the rulesets match on these full names. A caller that renames the job then fails loudly, instead of the gate waiting until its timeout.
- **A missing check counts as failing, never as skipped.** Branch protection and rulesets treat a skipped check as passing. So the job must always run and conclude `failure` when a portal has no test script or no test files. The portal must opt out explicitly (open question 1), not through a silent `if:` skip. This means `portal-tests.yml` needs a change: its current `if:` steps let the job conclude `success` with no tests.
- **Rulesets are the primary control.** Branch protection, or an org ruleset, requires these checks wherever the plan allows it. Repos that cannot require checks rely on the auto-merge gate, so the gate must check the same full names.

### 2. Grouping: a shared convention in each portal's `dependabot.yml`

Every portal uses the same group names:

- `dev-tooling`: build, bundler, CSS, lint, test, and type packages. Dev dependencies only.
- `prod-minor-patch`: production minor and patch updates.
- `actions`: GitHub Actions updates. Never auto-eligible.

Production majors are **not grouped**. Each one gets its own PR, because each usually needs its own migration. Groups are split by scope so one change is reviewed as one thing. Portal `dependabot.yml` files set the groups. The shared workflow reads the group name from the policy, and it never hard-codes a list of groups that could drift.

### 3. Policy as data, evaluated per dependency

A single policy file defines which updates are eligible. Portals use the shared default, and may override it in `.github/dependabot-policy.yml`.

Rules are evaluated top to bottom, first match wins. Deny rules come first, and the file ends with an explicit default deny, so an update type or ecosystem we did not anticipate fails closed:

```yaml
# Shared default. Evaluated per dependency entry; first match wins.
# Anything unmatched is not eligible.
rules:
  - ecosystem: github_actions
    eligible: false
  - update-type: major
    dependency-type: direct:development
    group: dev-tooling
    eligible: true
  - update-type: major
    eligible: false             # every other major: human review
  - update-type: [patch, minor]
    eligible: true
  - eligible: false             # default deny
```

Evaluation is per dependency, not per PR. The workflow reads `updated-dependencies-json` from `dependabot/fetch-metadata`, which lists every dependency in the PR, and checks each one against the rules. The PR is `eligible` only if every entry matches an eligible rule. One production major in a mixed group therefore blocks the whole PR. This avoids relying on the group-level `update-type` and `dependency-type` outputs, which do not describe a mixed group. The shape of `updated-dependencies-json` must still be confirmed on the pilot PR (see open question 2).

The shared workflow writes a single `eligible` output. Both the wait step and the merge step read that output. The duplicated `if:` in PR #5 goes away. Changing policy is a reviewed change to a data file.

### 4. Merge: only after checks pass on the exact commit

1. Wait for the required checks on `HEAD_SHA`.
2. Verify the PR head still equals `HEAD_SHA`.
3. Enable auto-merge bound to that SHA, for example `gh pr merge --auto --squash --match-head-commit "$HEAD_SHA" "$PR_URL"`. Confirm `gh` enforces the match when `--auto` is set.

Where required checks are enforced by a ruleset, GitHub re-gates auto-merge on any new push, so the race described above only applies to repos that cannot require checks. The SHA pin is kept as defense in depth. It does not replace the ruleset.

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
- An update that is out of policy gets the same label, with a reason ("production major", "GitHub Actions").
- Production majors stay as individual PRs. They are not grouped.
- A weekly report (issue or summary) lists open `needs-human` PRs per portal, so the backlog is visible.

## Rollout

1. **Pilot on one portal.** Add the build job to the reusable workflow, the policy file, and the grouping config. Confirm on a real Dependabot PR that: the build is gated, the check names are `ci / <job>`, the policy evaluates each entry in `updated-dependencies-json` to the expected `eligible` value, and the merge is bound to the checked SHA.
2. **Fix the gaps before widening.** Make missing checks fail, not pass (open question 1). Enable the ruleset or branch protection where the plan allows it. Confirm the `updated-dependencies-json` shape on a real grouped PR (open question 2).
3. **Roll out to the other portals.** Each portal gets the same caller workflow, group config, and policy reference. No per-portal logic.
4. **Review after one cycle.** Count auto-merged updates, reverts, and `needs-human` items. Adjust policy based on those numbers.

## Risks

- **A passing check does not prove safety.** Checks run the code, so a compromised release can pass them. Mitigated by keeping production majors manual and individual, Bun's default script blocking, cooldown, and eligibility limited to dev tooling.
- **Gaps in test coverage.** A portal with thin tests can auto-merge a broken update. Mitigated by the build gate and by counting no-tests as failing (open question 1).
- **Policy drift between portals.** Mitigated by the shared default and a single policy evaluation.
- **Check names change.** If a caller renames its job, gates can wait for names that never appear. Mitigated by the caller template fixing the job id, and by a timeout that fails closed.

## Open questions

Resolved:

- **No-tests behavior (decided).** A portal with no test script or files fails the gate and must opt out explicitly through its policy file. `dev-tooling` gets no exemption.
- **Policy ownership (decided).** The repo owner maintains the shared default policy. The team of three portal developers changes it through reviewed PRs.
- **Dependabot vs. Renovate (decided).** Stay on Dependabot. See Alternatives considered.

Open:

1. **`updated-dependencies-json` shape.** The per-dependency design assumes this output lists every dependency in a grouped PR with its update type and dependency type. Confirm on a real grouped PR during the pilot.
2. **Org ruleset or per-repo branch protection** for the required checks? Depends on plan limits for each repo. Determines how often the SHA pin is the only protection.

## Decision on PR #5

Do not merge PR #5 as written. It hard-codes group names, duplicates the eligibility condition, has the merge race, and has no build gate. Close it or convert it to draft. Its core idea, allowing major updates to dev tooling when checks pass, survives as one rule in the policy file.
