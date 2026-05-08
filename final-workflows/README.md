# Final workflows for kenobi-c2s — production-ready (semantic-release variant)

These are the **actual files** that would land in `crowdbotics/kenobi-c2s` if the team picked **semantic-release** as the release tool. Unlike the stub workflows at the root of this POC repo, these contain real ACR logins, helm `--set` matrix, secret/var wiring, and the tenant image build chain — copied 1:1 from the existing kenobi-c2s workflows where the behaviour is unchanged.

> **The decision of which tool to adopt — semantic-release or release-please — lives in COR-2624, not in this folder.** This directory captures what semantic-release adoption would look like in production, for that ADR's reference.

## What goes where

| File here | Goes to | Replaces |
|---|---|---|
| `release-and-dev.yml` | `kenobi-c2s/.github/workflows/release-and-dev.yml` | `azure-build-deploy-dev.yml` + `azure-build-on-tag.yml` |
| `promote.yml` | `kenobi-c2s/.github/workflows/promote.yml` | `azure-build-promote.yml` |
| `pr-title-lint.yml` | `kenobi-c2s/.github/workflows/pr-title-lint.yml` | (new — required) |
| `.releaserc.json` | `kenobi-c2s/.releaserc.json` (repo root) | (new) |
| `package.json` | merge with existing `kenobi-c2s/package.json` (or create new) | (semantic-release tooling deps only) |

## What gets deleted from kenobi-c2s

| File | Why |
|---|---|
| `.github/workflows/azure-build-deploy-dev.yml` | Folded into `release-and-dev.yml :: build-and-deploy-dev` job |
| `.github/workflows/azure-build-on-tag.yml` | Folded into `release-and-dev.yml :: build-versioned-image` job |
| `.github/workflows/azure-build-promote.yml` | Replaced by `promote.yml` (regex updated for bare semver) |
| `.github/workflows/azure-hotfix-ci.yml` | Replaced by `maintenance/X.x` release-branch model |

## Job graph in `release-and-dev.yml`

```
                    push to main / develop / maintenance/*
                                  │
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
      semantic-release        build-and-       (semantic-release
      (always; immediately    deploy-dev        only — the chore commit
       tags + cuts release    (always,          gets pushed back to
       + push-back commit)    parallel —        the release branch)
            │                  no `needs:`)
            ▼
      build-versioned-image
      (needs: semantic-release;
       if: new_release_published == 'true')
```

Same job-graph shape as the release-please variant. **Functional difference:** semantic-release's first job is non-gated (no Release-PR step) — the tag is created the moment the workflow runs.

## What stays unchanged from current kenobi-c2s

These workflows are orthogonal to release process:
- `auto-assign-reviewers.yml`, `auto-label-paths.yml`
- `azure-review-app.yml`, `azure-review-app-cleanup.yml`
- `azure-tenant-build.yml`
- `daily-status.yaml`, `weekly-roundup.yaml`, `monthly-roundup.yaml`
- `label-approved-prs.yml`, `llm-observability-coverage.yml`
- `migration_check.yaml`, `needs-attention.yml`
- `notify-slack-on-paths.yml`, `playwright-qa.yml`
- `pre-commit.yaml`, `run-tests.yml`, `security-scan.yaml`
- `update-skill-reference.yml`

## One-time repo settings to flip

```bash
# 1. Allow GitHub Actions to interact with PRs (for @semantic-release/github comments)
gh api -X PUT /repos/crowdbotics/kenobi-c2s/actions/permissions/workflow \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true

# 2. Squash-merge default — match the convention used by the release-please POC
gh api -X PATCH /repos/crowdbotics/kenobi-c2s \
  -f squash_merge_commit_title=PR_TITLE \
  -f squash_merge_commit_message=PR_BODY
```

## Critical: provision SR_GITHUB_TOKEN

semantic-release pushes `chore(release): X.Y.Z [skip ci]` commits back to the release branch. **With branch protection on `main` (production), the default `GITHUB_TOKEN` is rejected.** You must provision either:

- A **GitHub App installation token** (recommended — clean audit trail, scoped permissions, can be added to bypass-list)
- A **Personal Access Token** with `contents: write` (works, but tied to a person and rotates poorly)

Set as repo secret `SR_GITHUB_TOKEN`. The workflow falls back to `GITHUB_TOKEN` if not set (works only without branch protection).

This is a setup gap vs release-please, which can run with default `GITHUB_TOKEN` only.

## Branch protection on `main`

With semantic-release, branch protection on `main` becomes load-bearing — the merge-to-main IS the gate (no Release-PR review surface). Recommended setup:

- ✅ Require pull request before merging
- ✅ Require approvals (1+, codeowner-required for sensitive paths)
- ✅ Require status checks: `pr-title-lint / validate`, all CI checks
- ✅ Dismiss stale approvals on new commits
- ✅ Require conversation resolution before merging
- ✅ Restrict who can push (only the SR_GITHUB_TOKEN identity has bypass)

## Branches reference for `.releaserc.json`

The POC `.releaserc.json` configures three branch types:

```json
"branches": [
  "main",
  { "name": "develop", "prerelease": "rc" },
  { "name": "maintenance/+([0-9])?(.{+([0-9]),x}).x",
    "range": "${name.replace(/^maintenance\\//, '')}",
    "channel": "${name.replace(/^maintenance\\//, '')}" }
]
```

- **`main`** → stable releases (1.23.0, 2.0.0, ...)
- **`develop`** → prerelease channel cutting `X.Y.Z-rc.N` from develop pushes
- **`maintenance/X.x`** → release-branch hotfixes (e.g. `maintenance/1.x` cuts patches on the 1.x line, independent of main)

## Migration sequencing (recommended)

Same shadow-migration approach as the release-please POC suggests:

1. **PR-1**: Land `.releaserc.json`, `package.json` (merged with existing if present), and the three new workflows. Set up `SR_GITHUB_TOKEN`. **Don't delete old workflows yet.**
2. Push a tag matching the latest released version (e.g. `git tag <last-released-version> && git push --tags`). This anchors semantic-release's "last release" pointer; without it, semantic-release would default to 1.0.0 and produce backwards versions.
3. Cut one release end-to-end. Verify versioned image + tenant image build. Verify `promote.yml qa → production` works.
4. **PR-2**: delete the old `azure-build-*.yml` files. Add `pr-title-lint / validate` as required status check on `main`.

## Things that intentionally do NOT move into semantic-release

- **Helm chart `appVersion` (in project-deploy)** — different repo, different release cadence. If we want chart-bump-on-kenobi-release, it's a one-line follow-up workflow triggered from kenobi-c2s release events, not a semantic-release concern.
- **OnTenant ACR image copies** — already handled in the tenant-build jobs as in current kenobi-c2s. No change.
- **Slack notifications** — same `crowdbotics/github-actions/slack-notify@master` action, same secrets.

## What "feels" different to engineers

vs current kenobi-c2s:
- **PR titles must be Conventional Commits** — same as release-please.
- **Cutting a release** is now: merge a PR (the release happens automatically). No `git tag` needed.
- **Tag format** is bare semver (`1.2.3`) — same as release-please.

vs release-please:
- **No Release PR review step.** The merge-to-main IS the release. Branch protection on `main` becomes the only gate.
- **Push-back commits clutter `git log`.** Every release adds a `chore(release): X.Y.Z [skip ci]` commit.
- **Channel-based prereleases work natively.** `develop` branch with `prerelease: 'rc'` config cuts `X.Y.Z-rc.N` automatically. release-please requires manual `Release-As:` footers.
- **Auth setup is heavier.** SR_GITHUB_TOKEN must be provisioned with bypass-protection. release-please doesn't need this.

## Interaction with `scripts/generate_build_info.py`

Same as the release-please variant. The script reads `git describe --tags --exact-match HEAD` (primary) or `GITHUB_REF` (fallback). semantic-release's bare semver tags (`tagFormat: "${version}"`) produce identical `version` fields in `build.json` to release-please's bare tags. **No change to the script needed.**

`build-versioned-image` passes `GITHUB_REF=refs/tags/${{ needs.semantic-release.outputs.new_release_git_tag }}` to the docker build, matching the script's expected env-var format.
