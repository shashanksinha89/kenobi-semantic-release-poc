# Release Manager Guide — semantic-release POC

A hands-on walkthrough for reproducing every scenario in WRITEUP.md and *understanding* what each step does mechanically.

**Estimated time to run all 4 scenarios end-to-end: ~30 minutes.**

> **For the parallel release-please walkthrough**, see [shashanksinha89/kenobi-release-please-poc/RELEASE_MANAGER_GUIDE.md](https://github.com/shashanksinha89/kenobi-release-please-poc/blob/main/RELEASE_MANAGER_GUIDE.md). Same scenario structure; this guide explains the semantic-release variants.

---

## Part 1 — How this is working (read first)

### Mental model

```
PR opens          PR title lint runs      (Conventional Commits enforcement)
PR merges (squash) ────────────────────►  release-and-dev.yml fires:
                                          ├── semantic-release: parses commits since
                                          │   last tag, decides bump, IMMEDIATELY
                                          │   tags + creates GitHub Release +
                                          │   pushes chore(release):X.Y.Z back to main
                                          ├── build-versioned-image: gated on
                                          │   new_release_published == 'true'
                                          └── build-and-deploy-dev: parallel, always

(no Release PR step — release happens immediately on merge)
```

### Key contrasts vs release-please

- **No Release PR.** semantic-release tags + cuts the GitHub Release synchronously inside the workflow run. There is no human-reviewed staging PR.
- **Push-back commit.** Every release lands a `chore(release): X.Y.Z [skip ci]` commit on the release branch with the CHANGELOG + manifest bump. Adds noise to `git log`.
- **Manifest is written, not read.** `pyproject.toml` is bumped *as a side effect*. The source-of-truth for "what version are we on" is the most recent git tag.

### What types of commits qualify for a bump?

| Commit subject prefix | Bump type | Shows in changelog? |
|---|---|---|
| `feat:` | minor (1.2.0 → 1.3.0) | yes, "Features" |
| `fix:`, `perf:`, `refactor:`, `revert:` | patch | yes |
| `feat!:` or `BREAKING CHANGE:` footer | major (1.2.0 → 2.0.0) | yes, "BREAKING CHANGES" |
| `chore:`, `docs:`, `style:`, `test:`, `build:`, `ci:` | none | hidden |

The list is configured in `.releaserc.json :: plugins :: @semantic-release/commit-analyzer :: releaseRules`. Fully customisable.

### Override mechanisms (and what's missing vs release-please)

**semantic-release does NOT have a `Release-As: X.Y.Z` footer.** To force a specific version:

- **For a hotfix on an older line:** create a `maintenance/X.x` branch from a stable tag, configure it in `.releaserc.json :: branches`, push the fix → semantic-release patches the X.x line independently of main.
- **For arbitrary version override:** push a tag manually (`git tag X.Y.Z && git push --tags`), and the next semantic-release run will compute from there. Crude but works.
- **For "skip a release entirely":** use `[skip release]` in commit message — semantic-release's analog of release-please's `chore:` (which doesn't bump anyway).

There is no equivalent of release-please's `Release-As: 1.0.0` for "bless this prerelease as stable" without going through the channel/branch promotion flow.

### Auth requirement (load-bearing in production)

In the POC, default `GITHUB_TOKEN` worked because there's no branch protection on `main`. **In real kenobi-c2s with branch protection**, you must provide a PAT or GitHub App token via `secrets.SR_GITHUB_TOKEN` (the workflow looks for it with a fallback to `GITHUB_TOKEN`). The token needs:
- `contents: write` (to push the chore(release) commit + tag)
- Bypass-protection on `main` (because semantic-release pushes directly, not via PR)

This is the biggest setup gap vs release-please, which can run with default `GITHUB_TOKEN` only because release-please uses a Release PR which a privileged user merges manually.

---

## Part 2 — Setup

```bash
gh repo fork shashanksinha89/kenobi-semantic-release-poc --clone --remote
cd kenobi-semantic-release-poc

# Allow Actions to create PRs (needed for `develop` branch dev-preview PRs if you set them up later)
gh api -X PUT /repos/<your-username>/kenobi-semantic-release-poc/actions/permissions/workflow \
  -f default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true

# Set squash-merge to use PR title + body (matches the release-please POC convention)
gh api -X PATCH /repos/<your-username>/kenobi-semantic-release-poc \
  -f squash_merge_commit_title=PR_TITLE \
  -f squash_merge_commit_message=PR_BODY

# Create environments referenced by promote.yml
gh api -X PUT /repos/<your-username>/kenobi-semantic-release-poc/environments/qa
gh api -X PUT /repos/<your-username>/kenobi-semantic-release-poc/environments/production
```

Workflow files in place:

```
.github/workflows/
├── release-and-dev.yml   # name: "Release & Dev Deploy"
├── promote.yml           # name: "Promote to QA / Production"
└── pr-title-lint.yml     # name: "PR Title Lint"
```

---

## Part 3 — Scenario walkthroughs

### Scenario 1 — Push to main → dev auto-deploys, no release for non-bumping commits

```bash
echo "// tiny" > app/scratch.txt
git add -A
git commit -m "chore: Add scratch file for scenario 1 testing"
git push origin main
```

Verify ONE workflow run with three jobs:
- `semantic-release` — runs, decides "no release"
- `build-and-deploy-dev` — runs, deploys sha-tagged image
- `build-versioned-image` — skipped (new_release_published is empty)

```bash
sleep 15
gh run list --limit 1
gh release list  # should be unchanged
```

### Scenario 2 — Cut a release (no Release PR — auto-tag)

```bash
cat > app/new_feature.py <<'EOF'
def new_endpoint():
    return {"hello": "world"}
EOF
git add -A
git commit -m "feat(api): Add new endpoint for X

COR-1234"
git push origin main
```

Wait for the workflow:

```bash
sleep 30
gh run list --workflow release-and-dev.yml --limit 1
gh release list --limit 1
```

You should see:
- ✅ Tag `X.Y.Z` immediately (no human merge step)
- ✅ GitHub Release published
- ✅ A new commit on main: `chore(release): X.Y.Z [skip ci]` with CHANGELOG.md + pyproject.toml updates
- ✅ build-versioned-image ran (the gate fired)
- ✅ build-and-deploy-dev ran in parallel

**This is the most consequential difference from release-please.** No staging PR. No human gate. The version is final the moment your `feat:` commit lands.

### Scenario 3 — Promote a tagged release to QA, then production

Identical to release-please POC. `promote.yml` is byte-identical between the two POCs. All 6 sub-tests:

```bash
TAG=$(gh release list --limit 1 --json tagName --jq '.[0].tagName')

# 3a — qa happy path
gh workflow run promote.yml -f image_tag=$TAG -f environment=qa

# 3b — production happy path (stable)
gh workflow run promote.yml -f image_tag=$TAG -f environment=production

# 3c — bad shape
gh workflow run promote.yml -f image_tag="v$TAG" -f environment=qa  # fails

# 3d — non-existent tag
gh workflow run promote.yml -f image_tag=99.99.99 -f environment=qa  # fails

# 3e — prerelease to qa (allowed)
# Cut prerelease via develop branch first (see Scenario 4)

# 3f — prerelease to prod (blocked)
gh workflow run promote.yml -f image_tag=2.1.0-rc.1 -f environment=production  # fails
```

### Scenario 4 — Hotfix via maintenance branch (semantic-release's equivalent of release-please's `Release-As:` footer)

semantic-release does NOT have a `Release-As:` footer. To ship a hotfix on an older version line:

```bash
# 1. Add maintenance/* pattern to .releaserc.json branches (one-time)
# (already done in the POC's .releaserc.json — see file)

# 2. Cut maintenance branch from the relevant tag
git checkout -b maintenance/1.x 1.1.0
git push -u origin maintenance/1.x

# 3. Add the trigger + config commits if they aren't already on this branch
# (POC required cherry-picking these commits because the branch was cut from
# a tag that predated the maintenance config)

# 4. Push the hotfix commit
git commit -m "fix(security): Patch null-pointer in tenant resolver

COR-1235"
git push origin maintenance/1.x
```

semantic-release on maintenance/1.x will cut `1.1.1` (the next patch on the 1.x line), independent of whatever's on main.

**Footgun specific to semantic-release:** GitHub will mark `1.1.1` as **Latest** in the Releases UI, even though a higher version (e.g. 2.0.0) already exists on main. Misleading for end-users. release-please handles this correctly by default; semantic-release requires custom `@semantic-release/github` configuration to fix.

---

## Part 4 — Quick reference

| Question | Command |
|---|---|
| Did semantic-release run? | `gh run list --workflow release-and-dev.yml --limit 1` |
| What tag was last cut? | `gh release list --limit 1` |
| Did the build-versioned-image job run? | `gh run view <run-id>` |
| Did dev get a new sha-tagged build? | `gh run view <run-id>` (look for `build-and-deploy-dev` job) |
| What version does the manifest say? | `cat pyproject.toml | grep version` (note: it's a side-effect, not source-of-truth) |
| What channels are configured? | `cat .releaserc.json | jq .branches` |

### Mental-model table

| Action | Effect |
|---|---|
| Engineer opens regular PR with `feat:` title | PR title lint passes |
| Engineer merges that PR | semantic-release immediately tags + cuts release + pushes back chore commit |
| (No release manager merge step exists) | — |
| Release manager dispatches `promote.yml` with a tag | Helm upgrade in qa or production (semver/existence/prerelease guards) |
| Engineer pushes a commit to `develop` | semantic-release cuts `X.Y.Z-rc.N` prerelease from the `develop` channel |
| Engineer pushes to `maintenance/X.x` | semantic-release cuts a patch on the X.x line, independent of main |

**Key contrast with release-please:** there is no "human merges Release PR" step. The merge-to-main IS the release.
