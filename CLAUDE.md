# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo contains **only GitHub Actions workflows** (no application source code). Each workflow watches an upstream project's GitHub Releases and, when a new release tag appears that isn't yet in the corresponding GHCR registry, publishes an image to `ghcr.io/mengfei0053/proxy-agents-*`. CLIProxyAPI and 9router are built from the upstream tag. OmniRoute is copied from the upstream published image; a local Next.js build OOMs on `ubuntu-latest`.

There is no build/lint/test step — the unit of work is a workflow file under `.github/workflows/`.

## Workflows

| File | Upstream | Image | Platforms |
|------|----------|-------|-----------|
| `update-cliproxyapi-image.yml` | `router-for-me/CLIProxyAPI` | `ghcr.io/mengfei0053/proxy-agents-cliproxyapi` | amd64 + arm64 |
| `update-9router-image.yml` | `decolua/9router` | `ghcr.io/mengfei0053/proxy-agents-9router` | amd64 |
| `update-omniroute-image.yml` | `diegosouzapw/OmniRoute` | `ghcr.io/mengfei0053/proxy-agents-omniroute` | amd64 + arm64 (copied) |

Each runs on `schedule` (cron, UTC) at Beijing 00:00 = **UTC 16:00**, with minutes staggered (`:07`, `:17`, `:27`) to avoid overlap. All also expose `workflow_dispatch` for manual runs.

## Shared pattern (when adding a new upstream)

Every workflow follows the same skeleton; copy an existing one and adjust the `env` block:

1. `gh release view --repo <upstream> --json tagName -q .tagName` → latest tag.
2. Login to GHCR (`docker/login-action`).
3. `docker manifest inspect "$IMAGE:$TAG"` — **this is the "is it newer?" check.** Exit 0 = tag already published → skip; non-zero = build needed. (Existence of the tag is the comparison; there is no semver math.)
4. If building: `actions/checkout` with `repository:`/`ref: <tag>`, then `docker/build-push-action` pushing both `:<tag>` and `:latest`.

OmniRoute does not follow step 4. Upstream image tags drop one leading `v` from the release tag (`v3.8.52` → `ghcr.io/diegosouzapw/omniroute:3.8.52`, same tag on Docker Hub `diegosouzapw/omniroute`). The workflow prefers that public GHCR image, falls back to Docker Hub, and `skopeo copy --all` preserves the multi-arch index. Destination tags stay `:$TAG` (the release tagName, including `v`) and `:latest`.

Per-upstream differences that **must** be verified against the upstream repo before adding a workflow:
- **Platforms** — match what the existing GHCR image already publishes (`docker buildx imagetools inspect <image>:<tag>`). CGO-based Go builds (CLIProxyAPI) need QEMU for arm64; the 9router Node image is amd64-only.
- **Build args** — only pass `--build-arg`s the upstream `Dockerfile` actually declares. CLIProxyAPI takes `VERSION`/`COMMIT`/`BUILD_DATE`; 9router takes none.

## GHCR push authentication

Login uses `${{ secrets.GHCR_TOKEN || secrets.GITHUB_TOKEN }}`. Because the images are pre-existing **user-scoped** packages, `GITHUB_TOKEN` may lack write access — prefer setting a classic PAT with `write:packages` as the `GHCR_TOKEN` repo secret. Jobs declare `permissions: packages: write`.

## Operational notes

- Scheduled workflows only run from the **default branch** (`master`); a new workflow must be merged to `master` to ever fire on its cron.
- GitHub disables scheduled workflows after **60 days** of repo inactivity.
