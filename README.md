# kenobi-semantic-release-poc

POC for evaluating [semantic-release](https://github.com/semantic-release/semantic-release) as the release-automation tool for `kenobi-c2s`.

Tracks **[COR-2623](https://linear.app/corestory/issue/COR-2623/evaluate-semantic-release-for-kenobi-c2s)**, companion to [COR-2622](https://linear.app/corestory/issue/COR-2622/evaluate-release-please-for-kenobi-c2s) (release-please POC). Both spikes feed into **COR-2624** (final tool decision ADR).

**Status:** evaluation in progress.

---

## Where to start reading

| Doc | When to read |
|-----|--------------|
| **[WRITEUP.md](./WRITEUP.md)** | Full evaluation: TL;DR, answers to the 8 questions in the ticket, lifecycle scenarios tested, gotchas, recommendation. |
| **[RELEASE_MANAGER_GUIDE.md](./RELEASE_MANAGER_GUIDE.md)** | Step-by-step walkthrough for someone reproducing the scenarios. |
| **[final-workflows/README.md](./final-workflows/README.md)** | Production-ready workflow files for the actual kenobi-c2s migration if semantic-release wins. |

The **head-to-head comparison + final tool pick** lives in [COR-2624](https://linear.app/corestory/issue/COR-2624/), not here. This spike captures findings only.

---

## Companion repo

[shashanksinha89/kenobi-release-please-poc](https://github.com/shashanksinha89/kenobi-release-please-poc) is the parallel POC for release-please. Same seed version (1.22.0), same kenobi-c2s mirror, same 4 lifecycle scenarios. Direct apples-to-apples comparison.

---

## Repo layout

```
.
├── pyproject.toml              # release version source-of-truth (semantic-release bumps this)
├── package.json                # tooling-only (semantic-release is a Node CLI)
├── .releaserc.json             # semantic-release config: branches, plugins, tag format
├── .github/workflows/          # 3 stub workflows (echo placeholders for docker build / helm)
│   ├── release-and-dev.yml     # name: "Release & Dev Deploy"
│   ├── promote.yml             # name: "Promote to QA / Production"
│   └── pr-title-lint.yml       # name: "PR Title Lint"
├── final-workflows/            # production-ready files for kenobi-c2s
├── app/                        # demo Python files used to drive commits
├── WRITEUP.md
└── RELEASE_MANAGER_GUIDE.md
```

## Workflow structure

Same 3-file shape as the release-please POC final state:

| File | Trigger | Jobs |
|------|---------|------|
| `release-and-dev.yml` | push to main | `semantic-release` (always) + `build-versioned-image` (gated on `new_release_published`) + `build-and-deploy-dev` (parallel, no `needs:`) |
| `promote.yml` | manual dispatch | qa/prod helm upgrade with semver/tag/prerelease guards. **Identical to release-please POC** — promotion is tool-agnostic. |
| `pr-title-lint.yml` | pull_request | Conventional Commits enforcement. **Identical to release-please POC.** |

## Tag convention

Bare semver via `tagFormat: "${version}"` in `.releaserc.json`. Matches release-please POC + kenobi-c2s existing tag-history convention.

## Headline difference vs release-please

semantic-release **does NOT** create a Release PR. The flow is:

```
push feat: to main
        │
        ▼
release-and-dev.yml fires
        │
        ▼
semantic-release job runs
        │
        ├──► immediately tags 1.23.0 + creates GitHub Release
        ├──► commits CHANGELOG.md + pyproject.toml bump back to main
        │
        ▼
build-versioned-image fires (needs: + if: new_release_published == 'true')
build-and-deploy-dev fires in parallel (no needs:)
```

There is no human-merge step between "qualifying commit landed on main" and "tag exists in production." This is the most consequential evaluation question — see Q8 in WRITEUP.md.
