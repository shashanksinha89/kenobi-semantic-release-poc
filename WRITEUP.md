# semantic-release evaluation — Writeup

**Ticket:** [COR-2623](https://linear.app/corestory/issue/COR-2623/evaluate-semantic-release-for-kenobi-c2s)
**POC repo:** [shashanksinha89/kenobi-semantic-release-poc](https://github.com/shashanksinha89/kenobi-semantic-release-poc)
**Companion:** [shashanksinha89/kenobi-release-please-poc](https://github.com/shashanksinha89/kenobi-release-please-poc) (release-please POC)
**Date:** 2026-05-08

---

## TL;DR

**Works as advertised, but with two consequential differences from release-please:**

1. **No human gate before tag creation.** Every qualifying merge to `main` immediately cuts a tag, GitHub Release, and pushes a `chore(release): X.Y.Z [skip ci]` commit back to main — all on the same workflow run. There is no Release-PR step. The gate must move to "merge to main" via stricter branch protection if we want a human review point.

2. **Manifest is not the source of truth — git tags are.** semantic-release IGNORES `pyproject.toml` / `package.json` versions. It walks tag history to find "last release" and computes the next version from there. Side-effect: `pyproject.toml` is *written by* semantic-release as a `prepareCmd` step but is *not read* — so for a fresh repo with no tags, semantic-release defaults to `1.0.0` regardless of what `pyproject.toml` says.

The findings + recommendation here are spike-scope only. The head-to-head comparison + final tool pick lives in **COR-2624**.

---

## What the POC demonstrated end-to-end

8 commits, 5 tags cut, all 4 lifecycle scenarios validated:

| # | Action | Branch | Resulting tag | Notes |
|---|--------|--------|---------------|-------|
| 1 | `chore: Initial scaffold` | main | — | No release (chore: type) |
| 2 | `ci: Drop npm cache` | main | — | No release |
| 3 | `fix: Return 200 from healthcheck` | main | **1.0.0** | First release. **`pyproject.toml=1.22.0` ignored** — no prior tag, so semantic-release defaulted to 1.0.0. |
| 4 | `feat(api): Add tenant middleware` | main | **1.1.0** | Minor bump from 1.0.0 |
| 5 | `feat(api)!: Replace v1 endpoint with v2 contract` (with `BREAKING CHANGE:` footer) | main | **2.0.0** | Major bump |
| 6 | `feat(api): Develop-channel feature for RC test` | develop | **2.1.0-rc.1** | Prerelease via `branches: [{name: 'develop', prerelease: 'rc'}]` config |
| 7 | `fix(security): Patch null-pointer (hotfix on 1.x line)` | maintenance/1.x | **1.1.1** | Release-branch hotfix; correctly skipped the 2.0.0 breaking change |

All tags are bare semver (`tagFormat: "${version}"`). Matches kenobi-c2s' existing tag-history convention.

The 3-job workflow (`semantic-release` + `build-versioned-image` gated + `build-and-deploy-dev` parallel) ran cleanly across all scenarios. Same shape as the release-please POC for direct comparison.

---

## Answers to the 8 ticket questions

### 1. Squash-merge → PR title flow

**Same as release-please.** Squash commit takes the PR title; semantic-release parses it via `@semantic-release/commit-analyzer` with `conventionalcommits` preset. PR title lint contract is identical.

### 2. PR title lint — same action?

**Yes.** `amannn/action-semantic-pull-request@v5` works identically. The lint workflow file is byte-identical between the two POCs. **No tool difference here.**

### 3. COR-XXXX prefix — subject vs footer?

**Same answer as release-please:** put in the commit footer, not the subject. The POC commits used `COR-9999`, `COR-9998` etc. as footer. Changelog stays clean while ticket stays searchable via `git log --grep` and visible in PR body.

### 4. Multi-package versioning?

**Not tested empirically.** semantic-release supports monorepo via `semantic-release-monorepo` or per-package `cwd` configuration. It does NOT have a first-class `packages:` config like release-please. For kenobi-c2s today, single-package is fine. If split later, configuration is heavier than release-please's equivalent.

### 5. Auth model — GitHub App / PAT required?

**Default `GITHUB_TOKEN` worked in the POC.** semantic-release pushed back the `chore(release): X.Y.Z [skip ci]` commit (with CHANGELOG + version bump) to `main` using only the workflow's default token. **However:**

- This POC repo has NO branch protection on `main`. With branch protection, the default `GITHUB_TOKEN` is rejected — you'd need a PAT or GitHub App token with `contents: write` AND bypass-protection on `main`.
- semantic-release has to push commits AND tags. release-please only needs to merge a PR (which a privileged user did manually). The auth surface is wider for semantic-release.
- We left `SR_GITHUB_TOKEN` as an optional override in the workflow with a fallback to `GITHUB_TOKEN`. In production with branch protection enabled, that secret would be required.

**Verdict:** workable, but adds a setup step (provision PAT/App) that release-please doesn't need with the single-workflow `needs:` chain pattern.

### 6. Pre-release support — channels?

**Cleaner than release-please.** Native config:

```json
"branches": [
  "main",
  { "name": "develop", "prerelease": "rc" }
]
```

Pushing `feat:` to `develop` cut `2.1.0-rc.1` and tagged it as a prerelease in GitHub. Subsequent develop pushes increment `-rc.2`, `-rc.3`. Merging develop → main automatically cuts `2.1.0` stable.

This is semantic-release's strongest dimension. release-please can do prereleases via `Release-As: 2.1.0-rc.1` footer but it's manual; semantic-release does it via branch config and channels.

### 7. Migration from existing tag history?

**Reads tags directly.** Push the most recent kenobi-c2s tag (e.g., `19.0.2`) to a fresh repo before first run, and semantic-release will compute the next version from there. No manifest seeding required (because the manifest isn't the source of truth — see Q below).

**However:** the existing kenobi-c2s tag history starts back at `19.0.2`. After COR-2560's planned reset to 1.22.0, you'd push `1.22.0` as a tag and semantic-release would bump from there. Same outcome as release-please's `bootstrap-sha` + manifest pattern, just achieved differently.

### 8. The headline question — no-Release-PR / auto-tag-on-every-merge

**This is the real finding.**

In release-please, the flow is: merge PR → release-please updates Release PR → human reviews + merges Release PR → tag cut. **Two human steps**, with the Release PR as a gated review surface showing the version bump + changelog before it ships.

In semantic-release: merge PR → tag cut + GitHub Release + push-back commit. **One human step.**

What this means in practice:

- **No "release candidate review" surface.** Once a `feat:` lands on main, the version is irreversible. To roll back you'd cut a new patch with a `revert:` commit.
- **The merge-to-main is the gate.** This pushes the burden onto branch protection: required reviews, required status checks (PR title lint, tests, etc.) become load-bearing in a way they aren't with release-please. With release-please you can be sloppy on PR review and catch issues at the Release-PR stage. With semantic-release, that safety net is gone.
- **Faster release cadence.** Every qualifying merge is a release. Good if you want continuous delivery. Bad if you want to batch changes into a single human-reviewed bundle.
- **Push-back commits clutter history.** Every release adds a `chore(release): X.Y.Z [skip ci]` commit. Over time these are a non-trivial slice of `git log`.
- **GitHub Release `Latest` flag bug.** When we cut `1.1.1` on the maintenance/1.x branch (after 2.0.0 already existed), GitHub marked `1.1.1` as **Latest**. Misleading for end-users. semantic-release's `@semantic-release/github` plugin doesn't expose a `makeLatest: legacy` knob easily; release-please handles this correctly by default.

**For a team that today gates production via `azure-build-promote.yml workflow_dispatch`**, the loss of the Release PR review surface is a meaningful regression. The promotion gate at qa/prod still exists in both tools (we tested all 6 sub-tests of `promote.yml` work identically), but the *release cut* gate moves earlier in the funnel — into "should this PR merge" — which is a different cultural contract.

---

## Gotchas observed

1. **Manifest version is irrelevant.** `pyproject.toml=1.22.0` was ignored on first run. semantic-release wrote `1.0.0` to it via the `@semantic-release/exec` `prepareCmd` step. To start at a specific version, push a tag first; do not rely on the manifest.

2. **`actions/setup-node` requires a lockfile if you set `cache: npm`.** Drop the cache or commit `package-lock.json`. (POC dropped the cache — npm install runs fresh each time, ~10-15 second penalty.)

3. **Push-back commits.** semantic-release commits and pushes CHANGELOG + manifest bumps as `chore(release): X.Y.Z [skip ci]`. The `[skip ci]` prevents infinite loops but every release is a side-effect commit. Over a year, this is hundreds of commits.

4. **GitHub `Latest` release flag.** Cutting a maintenance-branch hotfix marks it `Latest` in the GitHub UI even though a higher version already exists. Confuses end-users.

5. **Branch-protection compatibility.** Default `GITHUB_TOKEN` cannot push to a protected `main` branch. With protection enabled, you must provide a PAT or GitHub App token. (Not tested in POC because branch protection wasn't set up; documented as a known constraint.)

6. **Hotfix via release branch requires upfront config.** The `branches:` config must include the maintenance branch pattern BEFORE you cut the branch. Otherwise the branch's `.releaserc.json` won't recognize itself as a release-eligible branch and the workflow won't trigger semantic-release. (Hit this in the POC; resolved by cherry-picking the trigger + config commits onto `maintenance/1.x`.)

---

## What this POC didn't test

- **Production branch protection compatibility.** POC ran with no branch protection on main. In real kenobi-c2s, `main` would have required reviews + status checks + likely admin enforcement, and the default `GITHUB_TOKEN` would fail. We documented this as a constraint but didn't empirically validate the PAT/App setup.
- **Multi-package monorepo.** Single-package only.
- **Long-tail behavior.** Did not run dozens of releases to observe push-back commit accumulation in `git log`.
- **Multi-month develop → main RC promotion flow.** We cut one RC; did not actually merge develop → main to observe the auto-promotion-to-stable behavior.

---

## What this means for the team

This spike is findings-only. The "should we adopt semantic-release vs release-please" decision lives in **COR-2624**. The contents of this writeup feed into that ADR alongside the release-please spike findings.

If the team wants to discuss before the ADR lands, the two key questions to surface are:

1. **Are we OK with releases being one merge step instead of two?** (i.e., losing the Release PR review surface in exchange for faster cadence)
2. **Are we ready to make merge-to-main the load-bearing gate?** (i.e., investing in branch protection rules + PAT/App provisioning to compensate for the lost Release-PR review)

If the answer to either is "no," semantic-release is a regression. If both are "yes," semantic-release's stronger prerelease-channel support and tighter automation may win out.
