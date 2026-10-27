# Release versioning

Company guidance for versioning GitHub repositories without full [git-flow](https://nvie.com/posts/a-successful-git-branching-model/).

**Opening a pull request?** Follow [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md). It is the step-by-step for trunk + release branches (Option A): a normal day, backports, tags, and the situations you will hit.

**Docker on GHCR?** Cut / Release PR / RC tags are in [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md). CI promote (retag RC → final, no rebuild): [DOCKER_RC_PROMOTION.md](./DOCKER_RC_PROMOTION.md).

[RELEASE_VERSIONING_OPTIONS.md](./RELEASE_VERSIONING_OPTIONS.md) is why we chose that model. This README is the one-page summary: what to do, per repo type.

## What we need

1. Finish the current release while development for the next one has already started.
2. Patch an older release and apply the same fix to every newer line that still needs it.

Git-flow can do both. It is more process than we need (`master` + `develop` + `feature` + `release` + `hotfix`). The same capabilities come from **one trunk, release branches only while a version is supported, tags, and cherry-picks**.

## Rule of thumb

Git branching follows **how many builds of this repo are live**, not the language.

| Live builds | Git model |
| --- | --- |
| Many (consumers pin versions) | **Option A:** `main` + `release/X.Y` + tags `vX.Y.Z`. Fix on `main`, cherry-pick onto release branches. |
| One (“always latest”) | **Trunk / GitHub Flow:** develop and tag on `main`. If you later must patch an old tag, branch from that tag (Option A on demand). |

A **tag** names one commit. A **GitHub Release** is created from that tag. Tags are not “on” a branch. Do not move tags. Do not tag today’s `main` as `v1.4.1` after `v1.5.0` already exists.

## Per repository type

| Repo | Git model | Compatibility (not Git) |
| --- | --- | --- |
| **npm packages** | Option A. Publish from tags. SemVer. | Breaking change = major. Support window (e.g. current + previous minor). |
| **NuGet packages** | Same as npm. | Same. |
| **Frontend (React)** | Trunk. Deploy from `main`. Short freeze branch only if QA needs it. | Must work with the API that is in production. |
| **Backend (REST + DB)** | Trunk if only latest runs in prod. Option A if customers run different *builds*. | Version **HTTP endpoints** (`/api/v1`, `/api/v2`) in the same codebase. Version the **schema** with expand → switch → contract migrations. |

Do **not** put API v1 on a Git branch and API v2 on another. Old and new endpoints live in the same backend deploy. Git `release/*` means “we still patch an old *build*.” API versions mean “old *clients* still call us.”

```
npm 1.4.2  ──installed by──►  frontend (latest deploy)
nuget 3.2  ──installed by──►  backend (latest deploy)
frontend   ──calls─────────►  backend /api/v1  (and later /api/v2)
backend    ──migrates──────►  database (expand → switch → contract)
```

## Option A in one picture

```
main:          *--*--*--*--*--*--*     ← next version (e.g. 1.5)
                      \
release/1.4:           *--*--*         tags: v1.4.0, v1.4.1
```

- Product fixes land on `main` first, then cherry-pick onto `release/1.4`.
- Version bumps and 1.4 changelog stay on the release branch.
- Do **not** merge `release/1.4` into `main` or into another release line. Merge means “take everything”; you only want the fix.
- Delete `release/1.4` when 1.4 is out of support. Keep the tags forever.

Cherry-picks can conflict. Resolve on the target branch’s terms, or re-implement the same intent. That is still cheaper than merging two version lines together.

## Patching an old version on trunk

If `v1.5.0` is already tagged and you need a **1.4.1 that must not include 1.5**:

```bash
git checkout -b release/1.4 v1.4.0
git cherry-pick -x <fix-already-on-main>
git tag -a v1.4.1 -m "Release 1.4.1"
git push origin release/1.4 v1.4.1
```

If you only run latest, skip this: fix on `main`, tag `v1.5.1`, deploy.

## Further reading

- [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md) — how to work Option A on GitHub, day to day and per situation.
- [RELEASE_VERSIONING_OPTIONS.md](./RELEASE_VERSIONING_OPTIONS.md) — options compared, why cherry-pick over merge, tag lifecycle, API vs DB vs Git versions, GitHub setup, [open source examples of Option A](./RELEASE_VERSIONING_OPTIONS.md#open-source-examples-of-option-a).
- [A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/) — git-flow (the thing we are simplifying).
- [Semantic Versioning](https://semver.org/)
- [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow)
