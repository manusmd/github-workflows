# GitHub workflows template

Single pipeline for Next.js frontends: CI (lint, typecheck, test, build) then optionally build and push Docker image to GHCR. **Deployment is not included**; use the image tag in your infrastructure repo with ArgoCD.

## Pipeline

One reusable workflow **Pipeline** with two jobs:

1. **ci** – Install deps, dependency audit (fails on high/critical), lint, typecheck, test, Next.js build. Node version from `.nvmrc`. Always runs.
2. **build-push** – Build Docker image, push to GHCR. Runs only when `push_image: true` (e.g. on push to `main`).

Image tag: `[version]-[branch]-[short sha]` (e.g. `1.0.0-main-a1b2c3d`). Full image: `ghcr.io/<owner>/<repo>:<tag>`.

## Requirements

- **App repo:** `Dockerfile` at repo root (and optionally `.dockerignore`). This repo only provides the workflow; the Dockerfile lives in each app repo.
- **Node version:** From `.nvmrc` if present; otherwise 20.
- **Scripts in `package.json`:** `lint`, `typecheck`, `test`, `build`. No-op if unused (e.g. `"typecheck": "echo ok"`).
- **GHCR:** `GITHUB_TOKEN` is used; infra must be able to pull from GHCR.

## Using this template

**Option A – Reusable (one file)**  
Add one workflow in your app repo. Replace `<owner>` with the org or user that owns this template repo.

`.github/workflows/pipeline.yml`:

```yaml
name: Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
permissions:
  contents: read
  packages: write
jobs:
  pipeline:
    uses: <owner>/github-workflows/.github/workflows/pipeline.yml@main
    with:
      push_image: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
```

- Push/PR to `main` runs CI. Push to `main` also builds and pushes the image. Use the image tag from the run summary in your infra repo for ArgoCD.

**Option B – Copy**  
Copy `.github/workflows/pipeline.yml` from this repo into your app repo. You own the YAML and can change it per repo.

## `build-and-deploy.yml` — ArgoCD deploy pipeline

A second reusable workflow for projects that keep their Helm `chart/` in the
app repo and let ArgoCD watch that repo directly (the `projects-infrastructure`
onboarding model). On push to `main` it builds the repo's `Dockerfile`, pushes
`ghcr.io/<owner>/<app-name>:<git-sha>`, then bumps `image.tag` in the repo's own
`chart/values.yaml` and commits it. Lives here (public) so public app repos can
call it — a public repo cannot use a reusable workflow from a private repo.

```yaml
jobs:
  build-and-deploy:
    permissions:
      contents: write
      packages: write
    uses: manusmd/github-workflows/.github/workflows/build-and-deploy.yml@main
    with:
      app-name: my-app
```
