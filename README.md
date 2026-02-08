# GitHub workflows template

Pipeline templates for Next.js frontends: CI, Docker build and push to GHCR. **Deployment is not included**; use the image tag in your infrastructure repo with ArgoCD.

## Pipelines

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **CI** | Push / PR to `main` | Lint, typecheck, test, Next.js build |
| **Build and push image** | Push to `main` | Build Docker image, push to GitHub Container Registry |

## Image tag for ArgoCD

Deployment is done manually in your infrastructure repo with ArgoCD. Use the image produced by **Build and push image**:

- **Where to see it:** Open the run in the [Actions](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/using-the-workflow-run-summary) tab. The **Image tag summary** step shows the full image and tag.
- **Tag format:** `[version]-[branch]-[short sha]` (e.g. `1.0.0-main-a1b2c3d`). Version from `package.json`, branch from the push ref; slashes in branch names become `-`.
- **Full image:** `ghcr.io/<owner>/<repo>:<tag>`

## Requirements

- **App repo:** Must contain a `Dockerfile` at the repo root (and optionally `.dockerignore`). This template only provides the workflows; the Dockerfile lives in each app repo.
- **Node version:** CI uses the version from `.nvmrc` if present; otherwise it defaults to 20.
- **Scripts in `package.json`:** `lint`, `typecheck`, `test`, `build`. Add no-op scripts if you don’t use one (e.g. `"typecheck": "echo ok"`).
- **GHCR:** The workflow uses `GITHUB_TOKEN`; the repo’s packages must be readable by whoever runs ArgoCD (e.g. pull from GHCR with a PAT or deploy key).

## Using this template

**Option A – Reusable workflow (no copy)**  
Add these two files under `.github/workflows/` in your app repo. Replace `<owner>` with the org or user that owns this template repo; use `@main` or `@v1` to pin a version.

`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  ci:
    uses: <owner>/github-workflows/.github/workflows/ci.yml@main
```

`.github/workflows/build-push.yml`:

```yaml
name: Build and push image
on:
  push:
    branches: [main]
permissions:
  contents: read
  packages: write
jobs:
  build-push:
    uses: <owner>/github-workflows/.github/workflows/build-push.yml@main
```

Workflows run in your app repo (checkout, `github.repository`, Docker push go to your repo). Updates to this template repo apply to all callers when they use `@main`, or pin with `@v1` for stability.

**Option B – Copy**  
Copy the contents of `.github/workflows` from this repo into your app repo. You own the YAML and can change it per repo.

**Then**  
Ensure your app repo has a `Dockerfile` at the repo root. Push to `main` to build and push. In your infra repo, point ArgoCD at the image tag shown in the workflow run summary.
