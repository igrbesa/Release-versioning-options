# Docker images: promote RC → final (GitHub Container Registry)

## Overview

**Day-to-day release process** (cut `release/X.Y`, Release PR with version `1.5.0`, RC tags, final tag) lives in [DEVELOPER_GUIDE.md](./DEVELOPER_GUIDE.md) — especially [§2.4](./DEVELOPER_GUIDE.md#24-pre-releases-release-candidates--for-qa) and [§8.1](./DEVELOPER_GUIDE.md#81-cut-the-next-release-while-main-moves-on).

This document is only the **GHCR / CI** piece: how to **build** on an RC tag and **promote** (retag) that image digest to the final tag without rebuilding.

**Rule:** what QA tested is what you ship. For RC → final with no code change, **promote the same image digest**. Do not run `docker build` again for the final tag.

| Git (see developer guide) | GHCR |
| --- | --- |
| Tag `v1.5.0-rc.1` (after Release PR set version file to `1.5.0`) | **Build** `…:1.5.0-rc.1` |
| Tag `v1.5.0-rc.2` (after QA fixes; version file still `1.5.0`) | **Build** `…:1.5.0-rc.2` |
| Tag `v1.5.0` after QA sign-off | **Promote** last RC → `…:1.5.0` (same digest) |
| Tag `v1.5.1` (patch) | **Build** new image |

You do not rename or delete the RC tag. Both tags remain; they point at identical bytes after promote.

---

## Why not rebuild?

A second `docker build` for `1.5.0` can differ from the RC even with the “same” Dockerfile (base tags moved, unpinned packages, non-reproducible layers). Promotion avoids shipping something QA never tested.

---


## Image naming on GHCR

Replace placeholders with your repo:

| Piece | Example |
| --- | --- |
| Registry | `ghcr.io` |
| Image | `ghcr.io/my-org/my-service` (must be **lowercase**) |
| RC tag | `1.5.0-rc.1` (match SemVer without the leading `v` of the git tag, or use `v1.5.0-rc.1` — pick one convention and stick to it) |
| Final tag | `1.5.0` |
| Immutable | `sha-a1b2c3d` (git commit short SHA) |

This guide uses **version tags without the leading `v`** on the image (`1.5.0-rc.1`, `1.5.0`) and keeps the leading `v` on **git** tags (`v1.5.0-rc.1`, `v1.5.0`), which matches common GHCR practice.

---

## Promote by hand (optional)

Useful for a one-off check. Prefer Actions for the real release.

```bash
# Login (PAT with read:packages + write:packages, or use Actions GITHUB_TOKEN)
echo "$GITHUB_TOKEN" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

# Inspect RC
docker buildx imagetools inspect ghcr.io/my-org/my-service:1.5.0-rc.1

# Promote: new tag, same digest (no rebuild)
docker buildx imagetools create \
  --tag ghcr.io/my-org/my-service:1.5.0 \
  ghcr.io/my-org/my-service:1.5.0-rc.1

# Verify
docker buildx imagetools inspect ghcr.io/my-org/my-service:1.5.0-rc.1 --format '{{.Manifest.Digest}}'
docker buildx imagetools inspect ghcr.io/my-org/my-service:1.5.0 --format '{{.Manifest.Digest}}'
# both must be identical
```

Use `docker buildx imagetools` (or `crane` / `skopeo`), not `docker pull` + `docker tag` + `docker push`, so multi-arch manifest lists stay intact.

---

## GitHub Actions

Two workflows cover the usual path:

1. **Build** when a git RC tag is pushed.
2. **Promote** when a final git tag is pushed (or via manual `workflow_dispatch`).

Put them under `.github/workflows/` in the component repository. Adjust `IMAGE_NAME` and permissions to match your org.

### 1. Build and push on RC (and on every version tag that needs a new image)

Build for **RC tags** and for **patch tags** (`v1.5.1`, …). Skip rebuild when the tag is a final `vX.Y.0` that only promotes a prior RC — that job is in workflow 2.

```yaml
# .github/workflows/docker-build-on-tag.yml
name: Build and push image on version tag

on:
  push:
    tags:
      - "v*.*.*"
      - "v*.*.*-rc.*"

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  # ghcr.io/owner/repo — must be lowercase
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Decide whether to build
        id: gate
        run: |
          TAG="${GITHUB_REF_NAME#v}"   # v1.5.0-rc.1 -> 1.5.0-rc.1
          echo "image_tag=$TAG" >> "$GITHUB_OUTPUT"
          # Build RCs and patches. Skip plain x.y.0 (that is promote-only).
          if [[ "$TAG" == *-rc.* ]]; then
            echo "build=true" >> "$GITHUB_OUTPUT"
          elif [[ "$TAG" =~ ^[0-9]+\.[0-9]+\.0$ ]]; then
            echo "build=false" >> "$GITHUB_OUTPUT"
            echo "Skipping build for $TAG — use the promote workflow."
          else
            echo "build=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Checkout
        if: steps.gate.outputs.build == 'true'
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        if: steps.gate.outputs.build == 'true'
        uses: docker/setup-buildx-action@v4

      - name: Log in to GHCR
        if: steps.gate.outputs.build == 'true'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Lowercase image name
        if: steps.gate.outputs.build == 'true'
        run: echo "IMAGE=${IMAGE_NAME,,}" >> "$GITHUB_ENV"

      - name: Build and push
        if: steps.gate.outputs.build == 'true'
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ steps.gate.outputs.image_tag }}
            ${{ env.IMAGE }}:sha-${{ github.sha }}
```

Alternatively, split into two workflows (`tags: ['v*.*.*-rc.*']` for RCs, and a separate patch filter) so the build job never runs on plain `vX.Y.0`.

### 2. Promote RC → final (no rebuild)

Run when the **final** git tag `v1.5.0` is pushed, or manually after QA sign-off.

```yaml
# .github/workflows/docker-promote-rc-to-final.yml
name: Promote GHCR image RC → final

on:
  push:
    tags:
      # Final minors/majors only, e.g. v1.5.0 — not v1.5.0-rc.1, not v1.5.1
      - "v*.*.0"
  workflow_dispatch:
    inputs:
      source_tag:
        description: "Existing GHCR tag (RC), without leading v"
        required: true
        default: "1.5.0-rc.1"
      target_tag:
        description: "New GHCR tag (final), without leading v"
        required: true
        default: "1.5.0"

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  promote:
    runs-on: ubuntu-latest
    # On tag push: skip if this is actually an RC tag somehow matching *.*.0
    if: ${{ github.event_name == 'workflow_dispatch' || !contains(github.ref_name, '-rc.') }}
    steps:
      - name: Resolve source and target tags
        id: tags
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "source=${{ inputs.source_tag }}" >> "$GITHUB_OUTPUT"
            echo "target=${{ inputs.target_tag }}" >> "$GITHUB_OUTPUT"
          else
            # git tag v1.5.0 → promote image 1.5.0-rc.1 → 1.5.0
            FINAL="${GITHUB_REF_NAME#v}"
            echo "target=$FINAL" >> "$GITHUB_OUTPUT"
            echo "source=${FINAL}-rc.1" >> "$GITHUB_OUTPUT"
          fi

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Lowercase image name
        run: echo "IMAGE=${IMAGE_NAME,,}" >> "$GITHUB_ENV"

      - name: Promote (same digest, new tag)
        run: |
          SRC="${IMAGE}:${{ steps.tags.outputs.source }}"
          DST="${IMAGE}:${{ steps.tags.outputs.target }}"
          echo "Promoting $SRC → $DST"
          docker buildx imagetools create --tag "$DST" "$SRC"

      - name: Verify digests match
        run: |
          SRC="${IMAGE}:${{ steps.tags.outputs.source }}"
          DST="${IMAGE}:${{ steps.tags.outputs.target }}"
          A=$(docker buildx imagetools inspect "$SRC" --format '{{.Manifest.Digest}}')
          B=$(docker buildx imagetools inspect "$DST" --format '{{.Manifest.Digest}}')
          echo "RC digest:    $A"
          echo "Final digest: $B"
          test "$A" = "$B"
```

**Convention in the tag-push path:** final `v1.5.0` always promotes `1.5.0-rc.1`. If QA needed `rc.2`, either:

- Promote manually with `workflow_dispatch` (`source_tag=1.5.0-rc.2`), or
- Change the resolve step to read the latest `*-rc.*` for that version from GHCR / a release note input.

### 3. How this hooks to the developer guide

Follow [§8.1](./DEVELOPER_GUIDE.md#81-cut-the-next-release-while-main-moves-on) through cut → Release PR → `v1.5.0-rc.N` → `v1.5.0`. At each RC git tag the **build** workflow runs; at `v1.5.0` the **promote** workflow runs (or `workflow_dispatch` if the last candidate was `rc.2`).

---

## Who does what (images only)

| Step | Who |
| --- | --- |
| Build workflow on RC / patch tags | CI |
| Promote RC → final | CI on final tag, or DevLead via `workflow_dispatch` |
| Confirm digests match before production deploy | DevLead / QA |

Roles for cut, Release PR, and tagging: [DEVELOPER_GUIDE.md §4](./DEVELOPER_GUIDE.md#4-who-does-what).

---


## Checklist

- [ ] RC image exists on GHCR and QA signed off **that digest**.
- [ ] Final promotion uses `imagetools create` (or crane/skopeo), not a new `docker build`.
- [ ] Digests of RC and final tags are identical.
- [ ] RC tag is left in place (audit trail).
- [ ] Patch releases (`v1.5.1`) get a **new** build, not a retag of `1.5.0`.
- [ ] Base images in Dockerfiles are pinned by digest when you *do* build, so RCs stay reproducible.

---

## Related

- Release candidates in git: [DEVELOPER_GUIDE.md §2.4](./DEVELOPER_GUIDE.md#24-pre-releases-release-candidates--for-qa)
- Cutting a release / tagging: [DEVELOPER_GUIDE.md §8.1](./DEVELOPER_GUIDE.md#81-cut-the-next-release-while-main-moves-on) and [§8.8](./DEVELOPER_GUIDE.md#88-tag-the-build-and-publish-the-github-release)
