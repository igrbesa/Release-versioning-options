# Release versioning options

This document compares ways to version independent releases of a GitHub repository. The goal is a process that is simpler than [git-flow](https://nvie.com/posts/a-successful-git-branching-model/), while still covering two real needs:

1. **Parallel releases.** You can freeze and finish the current release while development for the next one has already started.
2. **Fixes that travel forward.** A bug found in an older release can be patched there, and the same fix can be applied to every newer line that still needs it.

Git-flow can do both of those things. It is also heavier than most teams need: two immortal branches (`master` and `develop`), plus `feature/*`, `release-*`, and `hotfix-*`, each with strict merge rules. Vincent Driessen, who wrote the model, later said it is a poor fit for continuously delivered software, and still a reasonable fit only when you ship **explicitly versioned** software and support multiple versions in the wild.

If this repository is a **component / library / product with version numbers**, you are in that second category. You do not need the full git-flow ceremony. You need **one trunk, release branches, tags, and a backport rule**.

---

## What “versioning a release” actually means

Three different things get mixed together. Keep them separate:

| Concept | What it is | Typical GitHub artifact |
| --- | --- | --- |
| **Version number** | The public identity of a build (`1.4.2`) | SemVer in the package / changelog |
| **Immutable snapshot** | The exact commit that was shipped | Git tag `v1.4.2` + GitHub Release |
| **Maintenance line** | A branch you can still patch after ship | Branch `release/1.4` |

Tags are the source of truth for “what we shipped.” Branches are only needed while a line is still being changed (stabilizing a release, or issuing patches). Once a line is no longer supported, delete the branch; keep the tags forever.

Use [Semantic Versioning](https://semver.org/):

- **MAJOR** (`2.0.0`) — breaking change
- **MINOR** (`1.5.0`) — new functionality, backward compatible
- **PATCH** (`1.4.2`) — bug fix, backward compatible

That numbering is independent of the branching model. Every option below can use SemVer.

---

## Requirements mapped to Git mechanics

| Need | Git mechanism that satisfies it |
| --- | --- |
| Finish release N while starting N+1 | Cut a `release/N` branch; keep developing on `main` |
| Independent version lines | One long-lived branch per supported minor (or major) line |
| Fix an old release | Commit (or cherry-pick) onto that release branch, tag a patch |
| Propagate the fix to all newer releases | Land the fix on `main` first, then cherry-pick onto each supported release branch (newest → oldest) |
| Know exactly what shipped | Annotated tag + GitHub Release per version |

The rest of this document is about how much extra process you wrap around that core.

---

## Option A — Trunk + release branches (recommended)

This is the model Kubernetes, Node.js, Python, and most “we ship numbered versions” open-source projects use. It is git-flow with `develop` and `hotfix/*` removed.

### Branches

| Branch | Lifetime | Role |
| --- | --- | --- |
| `main` | Forever | Next unreleased work. Always the newest code. |
| `release/X.Y` | Until that line is unsupported | Stabilization + patches for version `X.Y` |
| `feature/...` | Days to a few weeks | Short-lived PRs into `main` (or into a release branch only for a backport) |

No second immortal branch. No special hotfix branch type. A hotfix is just a PR into the release branch that needs it, plus a backport/forward-port of the same change.

### Day-to-day

```
main:          *--*--*--*--*--*--*--*--*     ← next release (e.g. 1.5)
                      \
release/1.4:           *--*--*  tags: v1.4.0, v1.4.1
```

1. Develop on `main` via pull requests.
2. When 1.4 is feature-complete, branch `release/1.4` from `main`.
3. `main` immediately becomes 1.5 work. The two lines are independent.
4. Stabilize 1.4: version bump and changelog live only on `release/1.4`. Product bug fixes still land on `main` first, then are cherry-picked onto `release/1.4` (see below). Tag `v1.4.0` and publish a GitHub Release.
5. Later patches: same pattern — fix on `main`, cherry-pick onto `release/1.4`, tag `v1.4.1`, `v1.4.2`, …

### How stabilization commits reach `main`

They do **not** flow automatically. After the cut, `release/1.4` and `main` are independent histories. Git will not copy commits between them unless someone cherry-picks (or, worse, merges the whole branch).

Treat commits on the release branch as two kinds:

| Kind | Example | Goes to `main`? |
| --- | --- | --- |
| **Product fix** | crash, wrong result, security | Yes — this must exist on `main` or 1.5 ships with the same bug |
| **Release-only** | bump to `1.4.0`, 1.4 changelog, RC notes | No — `main` already has (or will have) its own version identity |

**Preferred direction: `main` → `release/1.4`.** Even while 1.4 is freezing, a bug found in the 1.4 candidate is fixed on `main` first, then cherry-picked onto `release/1.4`. That is how Kubernetes works after a release branch is cut: remaining release changes are cherry-picks *from* trunk *onto* the release branch. `main` already has the fix; there is nothing left to “bring back.”

```
main:            *--*--A--*--B--*     1.5 work continues; A and B are fixes
                      \        \
release/1.4:           *--A'--B'--    cherry-picks of A and B; then tag v1.4.0
```

**Cherry-picks can conflict. That is normal.** A cherry-pick is not a merge of histories. It takes the *diff* of one commit (how `main` changed those lines) and tries to apply that same diff onto `release/1.4`. Git needs the surrounding context to still look similar. If 1.5 work on `main` already edited the same file, the same function, or nearby lines, the context no longer matches and you get a conflict.

When it is usually clean:

- Right after `release/1.4` was cut, `main` has barely moved.
- The fix touches code that 1.5 has not changed.

When it often conflicts:

- `main` has already refactored, renamed, or rewritten the same area for 1.5.
- The fix on `main` uses an API, file, or helper that does not exist on 1.4.
- You squash-merged a large PR on `main` that mixed the bug fix with unrelated 1.5 work — the squash commit’s diff is then hard to apply onto the older tree.

What to do:

1. Open the cherry-pick as a PR into `release/1.4` (do not push a resolved cherry-pick straight to the protected branch).
2. Resolve the conflict **on the release branch’s terms**: keep 1.4 code for everything that is not the bug fix; apply only the fix.
3. If the 1.5 version of the fix cannot be applied (new API, different structure), **re-implement the same intent** on 1.4. The backport commit will differ from `main`. That is fine. Record the original SHA in the message (`cherry-pick -x` or `Backport of #123`).
4. If conflicts are constant in one area, that area has diverged; stop forcing identical diffs and treat 1.4 as a separate implementation of the fix.

Conflicts on a single-commit cherry-pick are still cheaper than merging the whole `release/1.4` branch into `main` (or the reverse). You resolve one fix, not a pile of version bumps and unrelated 1.5 features.

**Escape hatch: you already committed on `release/1.4`.** If someone fixed directly on the release branch (urgent, or the code on `main` already diverged), forward-port that one commit to `main`:

```
git checkout main
git cherry-pick -x <sha-from-release-1.4>
# open a PR to main if the repo requires PRs
```

Use `-x` so the commit message records the original SHA. If the cherry-pick conflicts because 1.5 already changed that code, resolve by re-implementing the *intent* of the fix, not by merging the whole `release/1.4` branch.

**Do not merge `release/1.4` into `main` when the release ships.** That is the git-flow “merge back to develop” step. It also brings version bumps, changelog noise, and any 1.4-only workaround. Cherry-pick the product commits; leave the rest.

A practical checklist at tag time: every product commit on `release/1.4` that is not already on `main` gets a cherry-pick PR to `main` before you call 1.4 done.

### How fixes propagate (the important rule)

**Always fix on `main` first, then backport.**

```
1. Open a PR to main with the fix. Merge it.
   → every future release already has it.

2. Cherry-pick that commit onto each supported release branch,
   newest first: release/1.4, then release/1.3, then release/1.2.

3. Each backport is its own PR. Tag a patch on each line that ships it.
```

Why newest-first: if 1.4 already has the fix and 1.3 does not, a customer who upgrades 1.3 → 1.4 must not re-introduce the bug.

Exception: the bug does not exist on `main` (code was rewritten). Fix only on the affected release branch. If a newer line still has the old code, cherry-pick there too.

Do **not** merge `release/1.3` into `release/1.4` into `main`. Cascade merges pull along release-only commits (version bumps, reverted features, old workarounds) and create chronic conflicts. Cherry-pick the fix; leave the rest.

### Why cherry-pick instead of merging the release branch

A **merge** says: “make this branch contain *everything* the other branch has.”  
A **cherry-pick** says: “apply *this one change* onto that branch.”

You want the second sentence. `release/1.4` is not a subset of `main` that you are ready to absorb. It is a mix of things `main` should get and things it must not:

```
release/1.4 unique commits
  ├── bump version to 1.4.0          ← must NOT land on main (main is 1.5)
  ├── 1.4 changelog / RC notes       ← must NOT land on main
  ├── 1.4-only workaround            ← must NOT land on main (1.5 rewrote that code)
  └── bug fix A (null check)         ← this is the only thing main needs
```

`git merge release/1.4` into `main` (or into `release/1.5`) cannot select A. It brings the whole pile. You then fight conflicts in `package.json` / version files, changelog, and any 1.4-only workaround — even though the only product question was “does `main` have the null check?”

Cherry-pick A. Version bumps stay on 1.4. If A conflicts, you resolve one fix, not the entire 1.4 vs 1.5 divergence.

Merging **older → newer** (`release/1.3` into `release/1.4`) has the same problem at a larger scale. 1.3 may contain a patch for code that 1.4 already deleted, a revert of a feature that 1.4 still ships, and its own `1.3.7` version bump. A cascade `1.3 → 1.4 → main` repeats that tax on every line.

Merging back is also the wrong operation if you already followed “fix on `main`, cherry-pick to release.” Then `release/1.4` already has a *copy* of the fix (new SHA). Merging the release branch into `main` tries to apply that copy on top of the original — duplicate hunks or a pointless conflict.

Git-flow merges the release branch back into `develop` because `develop` was defined as “everything that should ship next.” That only stays sane if `develop` has not grown a new version’s features yet. In Option A, `main` *has* — that is the whole point of cutting `release/1.4`. So “merge the freeze line into the next-version line” is mixing two products.

Conflicts still happen with cherry-picks (see above). They are scoped to one commit. Merge conflicts are scoped to every file the two branches changed since they diverged.

| | Cherry-pick the fix | Merge `release/1.4` → `main` |
| --- | --- | --- |
| What moves | One commit (or a small set you chose) | Every commit unique to 1.4 |
| Version file | Untouched on `main` | Conflict or wrong version on `main` |
| 1.4-only workaround | Left on 1.4 | Dragged into 1.5 |
| Conflict size | That fix vs current code | Whole 1.4 freeze vs whole 1.5 |
| Repeatable | Same recipe for 1.3, 1.4, 1.5 | Each merge makes the next merge harder |

### What this gives you vs git-flow

| git-flow | This model |
| --- | --- |
| `develop` + `master` | Only `main` |
| `release-*` for stabilization | Same idea: `release/X.Y` |
| `hotfix-*` from `master` | PR directly to `release/X.Y` (and to `main`) |
| Merge release into both `master` and `develop` | Tag on the release branch; `main` already moved on |
| `--no-ff` merge ritual | Normal GitHub PRs, squash or merge as the repo already does |

You keep the two capabilities you care about. You drop the dual-trunk bookkeeping.

### GitHub pieces that make this easy

- **Tags + GitHub Releases** for every `vX.Y.Z`.
- **Branch protection** on `main` and `release/*` (required reviews + CI).
- **Labels** such as `backport-to-release/1.4` on the original PR.
- A backport GitHub Action ([`korthout/backport-action`](https://github.com/korthout/backport-action) or similar) that, after merge to `main`, opens cherry-pick PRs onto the labeled release branches.
- Optional: [release-please](https://github.com/googleapis/release-please) (Google) to open a “release PR” that bumps the version and changelog from [Conventional Commits](https://www.conventionalcommits.org/). Merge that PR → tag + GitHub Release. It can run on `main` *and* on each `release/X.Y` (patch bumps on the release line).

---

## Option B — GitHub Flow (too simple for your requirements)

[GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow): one branch (`main`), short-lived feature branches, every merge is potentially a production deploy. GitHub itself works this way.

| Requirement | Fit |
| --- | --- |
| Parallel “finishing 1.4 while starting 1.5” | Poor. There is only one line. |
| Patch an old version | Poor. Old versions are just tags; you would have to recreate a branch from the tag ad hoc. |
| Propagate fixes | Unnecessary if only one live version; insufficient if customers stay on old versions. |

Use this if the component is only ever consumed at “latest” (internal service, continuously deployed app). Do **not** use it if customers pin versions or you must patch 1.3 after 1.4 has shipped.

---

## Option C — Full git-flow (more process than you need)

Classic git-flow ([nvie.com](https://nvie.com/posts/a-successful-git-branching-model/)):

- Immortal: `master` (production) and `develop` (integration)
- Temporary: `feature/*`, `release-*`, `hotfix-*`

It does support parallel releases and hotfixes. The cost is:

- Every release is merged twice (`master` and `develop`).
- Hotfixes have their own branch type and merge matrix (including “if a release branch exists, merge the hotfix there instead of `develop`”).
- `master` and `develop` drift; people forget which one to branch from.
- Tooling (`git-flow` CLI) is extra surface area on top of GitHub PRs.

If you already feel this is too complicated, it is. Option A is the same product model with half the moving parts.

---

## Option D — Trunk-based development + feature flags

Google, Meta, and Amazon run **trunk-based development**: almost everyone commits to `main` (or to branches that live hours, not weeks). Incomplete work is hidden behind feature flags. Releases are tags (or very short-lived release branches cut at the last moment).

This is the DORA “elite” delivery model. It is excellent for a **single production** that you control.

It is a weak default for a **versioned component** unless you also add release branches for every line you still support. At that point you have reinvented Option A, plus you now also need a mature feature-flag platform.

Use trunk-based ideas *inside* Option A: keep feature branches short, merge to `main` often, hide unfinished work with flags if the component is deployed as a service. Do not use pure TBD as the release model if you must patch old versions.

---

## Option E — GitLab Flow (environment branches)

GitLab Flow is GitHub Flow plus either:

- **Environment branches** (`staging`, `production`) that you merge *into* as a promotion gate, or
- **Release branches** (`2-3-stable`) with cherry-picks from `main`.

The environment-branch variant is for “merge ≠ deploy” (promote the same commit through staging → prod). That solves deployment, not multi-version support.

The release-branch variant is essentially Option A. If you need both staging promotion *and* versioned lines, keep Option A for versions and use GitHub Environments / a `staging` deploy from `main`, not a long-lived `staging` branch.

---

## How large companies actually do it

There is no single “big tech branching strategy.” There are two clusters, based on **how many versions are live**.

### Continuously delivered products (one live version)

| Org | Practice |
| --- | --- |
| **Google** | Trunk-based, giant monorepo, almost all work on one mainline. Releases are snapshots / cherry-picks onto a release branch at cut time. Incomplete features behind flags. |
| **Meta** | Same idea: one trunk, very short branches, flags, continuous deployment of the web/app trunk. |
| **Amazon** | Many small services, each deploying from mainline independently. |
| **GitHub** | GitHub Flow: `main` is deployable; feature PRs; deploy from `main`. |

These companies rarely “fix 1.3 after 1.4 shipped” for the same product, because they only run one version. A fix goes to trunk and rolls forward.

### Versioned platforms and components (several live versions)

| Org / project | Practice |
| --- | --- |
| **Kubernetes** | Develop on `master`. At RC, cut `release-X.Y`. Further changes to that line are **cherry-picks from master**, newest supported branch first. Patch releases `vX.Y.Z` are tagged on the release branch. Support window is about 12 months. See [cherry-picks.md](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-release/cherry-picks.md). |
| **Node.js** | `main` for current development. `vN.x` release lines. LTS lines stay open for years. Fixes land on `main`, then are backported with labels / a backport process. |
| **Python** | `main` plus `X.Y` branches. Security/bugfix releases are tagged on the maintenance branch. |
| **Chrome / Chromium** | **Release trains**: trunk is always open. At a schedule, a snapshot is promoted through canary → beta → stable. Fixes are cherry-picked onto the branch that is currently in that channel. Multiple channels exist at once (exactly “finishing this release while the next one is already in development”). |
| **Microsoft (.NET, VS, Windows)** | Release branches per version / servicing line. Servicing (patches) on the old branch; new work on mainline. Backport by cherry-pick, not by merging old → new. |
| **Linux kernel** | `master` (Linus) plus stable trees (`linux-6.x.y`) maintained by cherry-picking from mainline. Explicitly **not** merging stable back into mainline. |

The shared pattern for versioned software is Option A:

> One trunk. A release branch per supported version. Fix on trunk. Cherry-pick back. Tag every ship.

Git-flow is a 2010 packaging of that idea with extra immortal branches. Big companies that ship versions did not keep the extra branches; they kept the release lines and the cherry-picks.

---

## Comparison

| | A. Trunk + release branches | B. GitHub Flow | C. Git-flow | D. Pure trunk + flags |
| --- | --- | --- | --- | --- |
| Finish N while starting N+1 | Yes — `release/N` vs `main` | No | Yes | Only if you add release branches |
| Patch an old release | Yes | No (ad hoc from tag) | Yes | No (unless release branches) |
| Propagate fix to newer lines | Cherry-pick (or already on `main`) | N/A | Merge hotfix to `develop` | Automatic on trunk |
| Number of long-lived branches | 1 + number of supported lines | 1 | 2 + release/hotfix | 1 |
| Cognitive load | Low–medium | Lowest | High | Low, but flags are the cost |
| Fits a versioned component | **Yes** | No | Yes, overkill | Only with extra release lines |
| Fits a continuously deployed app | Possible, often overkill | **Yes** | Poor | **Yes** |

---

## Recommended policy (concrete)

Adopt **Option A** with the following conventions.

### Naming

```
main                 # next version
release/1.4          # maintenance for 1.4.x
v1.4.0, v1.4.1       # tags (always v-prefix, always SemVer)
```

Do not use `master` as a second production branch. Production is a **tag**, not a branch.

### Support window

Decide how many lines you patch, and write it down. Example:

- Current minor: full fixes
- Previous minor: security + critical bugs only
- Older: no patches; customers upgrade

Without a window, “propagate to all new releases” becomes an unbounded cherry-pick tax.

### When to delete a release branch

Yes. `release/X.Y` is temporary. Delete it when you will not ship another `X.Y.Z`.

| Still exists | Why |
| --- | --- |
| `release/1.4` while 1.4 is supported | You may still tag `v1.4.1`, `v1.4.2` |
| Tags `v1.4.0`, `v1.4.1`, … | Forever. This is the record of what shipped |
| GitHub Release for each tag | Forever (or until you choose to archive notes) |

Do **not** delete the branch at `v1.4.0`. That is when maintenance *starts*. Delete it when the support window for 1.4 ends (for example when 1.6 ships and you only patch current + previous minor).

Deleting the branch does **not** delete tags. In Git they are different refs (`refs/heads/release/1.4` vs `refs/tags/v1.4.0`). The tagged commits stay reachable through the tags, so they are not garbage-collected. GitHub Releases stay too — they are attached to the tag, not to the branch.

After deletion, `git checkout v1.4.2` still works. If you ever need an emergency patch for an unsupported line, recreate the branch from the last tag (`git checkout -b release/1.4 v1.4.2`), patch, tag `v1.4.3`, then delete the branch again.

`main` is never deleted.

### Fix workflow

```
Bug found on 1.3 (1.4 is current, 1.5 is in development on main)

1. PR → main          (so 1.5+ is fixed)
2. Label: backport-to-release/1.4, backport-to-release/1.3
3. Automation opens cherry-pick PRs (or do it by hand)
4. Merge each backport PR
5. Tag v1.4.1 and v1.3.7, publish GitHub Releases
```

If the change does not apply cleanly, do not force it. A small manual adaptation on the old branch is normal; keep the *intent* of the fix, not necessarily the identical diff.

### Cutting a release

```
# 1.4 is feature-complete on main
git checkout -b release/1.4 main
# bump version to 1.4.0-rc.1 if you use RCs (release-only commit; do not cherry-pick to main)

# blockers found during QA:
#   1. PR the fix to main
#   2. cherry-pick that commit onto release/1.4
# do not merge release/1.4 into main

# when ready
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin release/1.4 v1.4.0
# create GitHub Release from the tag
```

From this moment, `main` may already contain 1.5 work. That is expected. Product fixes from the 1.4 freeze are already on `main` if you followed `main` → release. If anything was committed only on `release/1.4`, cherry-pick those product commits to `main` before you forget.

### What not to do

- Do not merge `release/1.4` back into `main` as a routine step. If a fix exists only on the release branch, cherry-pick it to `main`.
- Do not keep a `develop` branch “because git-flow has one.” It duplicates `main`.
- Do not create `hotfix/*` branches as a required type. A hotfix is a PR with a `patch` version bump.
- Do not treat `main` as “whatever is in production.” Production is the latest tag on each supported line.

---

## Suggested GitHub setup (minimal)

1. Default branch: `main`.
2. Protect `main` and `release/*`: required PR, required CI, no force-push.
3. Tag format: `vX.Y.Z`.
4. Changelog: either Conventional Commits + release-please, or a manually maintained `CHANGELOG.md` updated on the release PR.
5. Optional automation, in this order of value:
   1. CI on every PR to `main` and `release/*`
   2. Backport action driven by labels
   3. release-please (or similar) for version bump + GitHub Release

You can start with (1)–(3) only. Add automation when the second supported line exists and cherry-picks become regular.

---

## Where tags live (trunk), and how to patch an older version

A **tag is not “on” a branch.** It is a name for one commit. A GitHub Release is created from that tag. The commit is often reachable from `main` (so people say “we tagged main”), but Git does not attach the tag to the branch.

On a trunk / GitHub Flow repo, the usual sequence is:

```
main:  *----*----*----*----*----*----*
            ^         ^
         v1.4.0    v1.5.0     ← tags point at these commits
                                main then keeps moving
```

- `v1.4.0` and `v1.5.0` stay on those exact commits forever. Do not move a tag.
- New commits after `v1.5.0` are **not** part of 1.5. They become 1.5.1 or 1.6.0 when you tag them.
- The GitHub Release for 1.5.0 is that tag, not “whatever `main` is today.”

### You cannot tag today’s `main` as a fix for 1.4

After `v1.5.0` exists, `main` already contains 1.5 work. A fix merged to `main` sits on top of 1.5:

```
main:  *----*----*----*----*----F----*
            ^         ^         ^
         v1.4.0    v1.5.0    this commit is 1.5 + fix
                             NOT a valid v1.4.1
```

Tagging `F` as `v1.4.1` would ship 1.5 features under a 1.4 number. Wrong artifact, wrong SemVer.

### What to do instead — two cases

**Case 1 — you only run latest** (typical frontend / SaaS backend). “Previous version” is just what is in production. You do not need a 1.4 artifact.

- If prod is already 1.5: fix on `main`, tag `v1.5.1` (or deploy `main` without fussing over numbers), ship that.
- If prod is still 1.4 and `v1.5.0` is tagged but **not** deployed: either ship 1.5 including the fix (roll forward), or use Case 2 if 1.5 must not go out yet.

Rolling forward is the trunk default. That is why pure trunk is enough when only one build is live.

**Case 2 — you need a 1.4.1 that must not include 1.5.** Branch from the old **tag**, not from current `main`. This is Option A appearing on demand:

```
git checkout -b release/1.4 v1.4.0
# prefer: fix already on main → cherry-pick it here
git cherry-pick -x <sha-of-F>
# if it conflicts, re-implement the same intent on 1.4 code
git tag -a v1.4.1 -m "Release 1.4.1"
git push origin release/1.4 v1.4.1
```

```
main:           *----*----*----*----*----F----*     v1.5.0 is here; F is the fix
                     \
release/1.4:          *----F'                         tag v1.4.1 here
                      ^
                   v1.4.0
```

- `v1.4.1` lives on the release-branch commit `F'`, **not** on `main`.
- The GitHub Release for 1.4.1 is created from tag `v1.4.1`. GitHub does not require that tag to be on the default branch.
- If `F` is not on `main` yet, land it on `main` first, then cherry-pick. If you only fixed on `release/1.4`, cherry-pick `F'` to `main` so 1.5+ stays fixed.

If 1.4 will not be patched again, delete `release/1.4` after tagging. If it will, keep the branch — you have just adopted Option A for that line. That is the intended escape hatch: trunk until you need a second live build, then grow a release branch from the tag.

### Do not

- Move `v1.5.0` to a new commit (“fix the tag”). Tags are immutable; GitHub Releases would lie about what was shipped.
- Tag current `main` as `v1.4.1` after `v1.5.0` exists.
- Expect `main` to produce two different version lines by itself. One branch, one newest line. Older lines need a branch cut from their tag.

---

## Applying this to our repository types

The Git branching model does **not** change by language (React vs .NET vs npm). It changes by one question: **how many versions of this artifact are live at the same time?**

A second, separate question is **compatibility between artifacts** (frontend ↔ API, API ↔ database, app ↔ package). That is not solved by extra Git branches. It is solved by contracts: SemVer for packages, API versioning + expand/contract for the backend, additive migrations for the database.

Do not put “API v1” on `release/1` and “API v2” on `release/2` as a substitute for versioning endpoints. Multiple API versions usually live **in the same backend deploy**. Git release branches are for “we still ship patches of an old *build*.” API versions are for “old *clients* still call us.”

### Decision per repo type

| Repo type | How many versions are live? | Git model | Extra contract (not Git) |
| --- | --- | --- | --- |
| **npm packages** | Many. Frontends pin `1.4.2` while you publish `1.5.0`. | **Option A** (full): `main` + `release/X.Y` + tags. SemVer is the public API. | Breaking change = major bump. Keep a support window (e.g. current + previous minor). |
| **NuGet packages** | Same as npm. Other services / apps pin versions. | **Option A** (full). Same SemVer + backport rules. | Same. Document breaking changes. Prefer additive APIs. |
| **Frontend (React)** | Usually **one**: the browser always loads latest. | **Trunk / GitHub Flow.** Tags optional for “what is in prod.” Cut a short `release/X.Y` only if you freeze a train (QA) while `main` continues. Delete it after ship unless you must hotfix an old deployed bundle (rare for a SPA). | Must tolerate the **currently deployed API**. During backend rollout, frontend may need to speak both old and new response shapes for a while. |
| **Backend (REST + DB)** | Usually **one process** in prod (SaaS). Many **API versions** inside that process. Many **DB schema versions** over time, but one live schema. | **Trunk / GitHub Flow** if you only run latest. **Option A** if customers actually run different backend *builds* (on-prem, slow rollout, N-1 canaries you must patch). | **API compatibility** and **DB expand/contract** (below). These are mandatory even if you never create a `release/*` branch. |

If a frontend or backend is shipped to customers who **do not auto-update** (installed app, on-prem appliance, mobile wrapper with a frozen bundle), treat that repo like a package: Option A, because multiple builds are live.

### npm and NuGet — this is the Option A home turf

Consumers choose when to upgrade. That is exactly “finish 1.4 while 1.5 is in development” and “patch 1.4 after 1.5 exists.”

- Publish from tags: `@company/ui@1.4.2` / `Company.Lib 1.4.2`.
- `main` is the next minor/major.
- `release/1.4` exists until 1.4 is out of support.
- Fix on `main`, cherry-pick to `release/1.4`, publish a patch.
- **SemVer is the compatibility contract** with the frontend (or with other .NET apps). A breaking change in a package is a major version, not a new Git branching model.

Frontend (or a backend) upgrades the package in *its* repo with a normal PR. Do not try to keep package versions in lockstep with backend release numbers. They are different artifacts.

### Frontend — usually no long-lived release lines

A typical React SPA has one production. A bugfix is: merge to `main`, deploy. You do not patch “frontend 1.3” while “frontend 1.4” is live, because users are not on 1.3.

Use Option A’s *ideas* only as needed:

- Short-lived `release/2026.08` (or `release/1.4`) if QA must freeze a candidate while `main` already has the next feature. After production deploy, delete the branch (tags can remain).
- Hotfix: if `main` has already moved and you cannot deploy it, branch from the production tag, fix, deploy, **and cherry-pick the fix to `main`**. Then delete the hotfix branch.

What the frontend *must* handle carefully is **not** Git lines, it is **API compatibility**:

- Frontend N must work with the backend that is in production today.
- If backend deploys a new field, frontend can ignore it (backward compatible).
- If backend removes or renames a field, frontend must be updated **before** that backend deploy, or the backend must keep the old field until the new frontend is out (expand/contract).
- Independent deploys work only if the API is backward compatible across at least one release (backend N works with API N and API N−1, or the reverse — pick a rule and write it down).

A practical rule: **frontend may deploy anytime; backend breaking changes may deploy only after all live frontends understand them.** Or: **backend is backward compatible with the previous backend.** Choose one; do not require a joint lockstep release unless you have to.

### Backend — Git version ≠ API version ≠ database version

Three clocks. Do not merge them into one branch strategy.

| Clock | What it versions | Typical mechanism |
| --- | --- | --- |
| **Git / artifact** | The deployed binary (`backend 5.1.0`) | Tags; Option A only if multiple *binaries* stay live |
| **HTTP API** | The contract browsers and other clients call | `/api/v1/...` vs `/api/v2/...` (or headers), **in the same codebase** |
| **Database** | Tables and columns | Ordered migrations (expand/contract) |

**API versioning (do this in code, not with Git branches):**

- Additive change (new optional field, new endpoint): stay on the same API version. Old frontends ignore unknown fields.
- Breaking change (remove field, change meaning, change URL, change error shape): introduce `/api/v2` (or the next header version). Keep `/api/v1` running until every live client is gone. Then delete v1 in a later backend release.
- Never deploy a breaking v1 change and a frontend that needs it in an unordered way. Run both API versions during a deprecation window.
- Do **not** maintain `release/api-v1` as a long-lived Git line whose only purpose is “the old API.” That duplicates the whole backend. Old API handlers live next to new ones on `main`.

**Database (this is the extra care the backend needs):**

- Migrations are **forward**. You almost never “patch schema 1.3” as a parallel line the way you patch an npm 1.3.
- **Expand/contract** so the running API (including old `/v1` handlers) never breaks mid-rollout:
  1. Expand: add column/table, nullable, backfill. Deploy. Old and new code both work.
  2. Switch code to use the new shape (still compatible with old rows if needed).
  3. Contract: later, drop the old column when nothing reads it.
- A backend deploy must be compatible with the **current** schema and, during a rolling deploy, with the **previous** app version still running (two app versions, one schema — so the schema change must be additive).
- Rollback of the app is possible only if the last migration did not destroy data the old app needs. That is why contract (drop column) is a separate later release.

If you run **one** production backend, GitHub Flow + API versions + expand/contract is enough. `release/*` branches do not make the database safer.

If you run **several** backend builds (customer A on 5.0, customer B on 5.1), then that repo *also* needs Option A: patch 5.0 on `release/5.0`, cherry-pick to `main`. Migrations on an old line are painful; prefer pushing customers forward, and only backport data-safe fixes.

### How the four repos fit together

```
npm 1.4.2  ──installed by──►  frontend (latest deploy)
nuget 3.2  ──installed by──►  backend 5.1.0 (latest deploy)
frontend   ──calls─────────►  backend /api/v1  and later /api/v2
backend    ──migrates──────►  database (expand → switch → contract)
```

Recommended defaults for this company:

1. **npm + NuGet:** Option A, SemVer, published from tags, support window written down. This is where “fix old release, propagate forward” actually happens.
2. **Frontend:** deploy from `main` (or a short freeze branch). No long-lived `release/1.4` unless you truly have multiple live bundles.
3. **Backend:** deploy from `main`. Version **endpoints**, not Git branches. Version the **schema** with additive migrations. Use Option A on the backend only if multiple backend *builds* stay in production.
4. **Compatibility rule** (write this once for the org):
   - Packages: SemVer. Breaking = major. Dependents upgrade on their own schedule.
   - HTTP: old API version stays until clients are gone. New frontend may require a minimum API version; new backend must not break the previous frontend until that frontend is gone.
   - DB: expand/contract; never a breaking schema change in the same release as a breaking API change unless you can take downtime and lockstep all clients.

The branching strategy stays Option A where multiple *builds* are live, and collapses to trunk where only *latest* is live. API and database compatibility are additional rules on the backend (and on how frontend is deployed), not extra branch types.

---

## Decision in one paragraph

If this repository ships **numbered, independently consumed versions** (npm, NuGet, or an app customers do not auto-update), use **trunk + `release/X.Y` branches + tags**, fix on `main`, cherry-pick backward. That is what Kubernetes, Node.js, and most component teams do, and it is git-flow without `develop` and without a special hotfix branch type.

If this repository is **only ever deployed as “latest”** (typical React frontend, typical SaaS backend), use GitHub Flow or trunk-based development and skip long-lived release branches until the first time you must patch an old *build*. For the backend, still version **HTTP APIs** and **database migrations** as contracts inside the same codebase — that is not a Git branching problem.

Start with that split. It matches both stated requirements, stays inside normal GitHub PR practice, and can grow a support window and backport automation later without changing the mental model.
