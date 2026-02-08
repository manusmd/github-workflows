# Pipeline usage guide

How to use the [manusmd/github-workflows](https://github.com/manusmd/github-workflows) pipeline template in a project (e.g. Next.js app). The pipeline runs CI and builds/pushes a Docker image to GHCR; deployment is done manually via ArgoCD in your infrastructure repo.

---

## What the pipeline does

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **CI** | Push / PR to `main` | Lint, typecheck, test, Next.js build. Node version from `.nvmrc`. |
| **Build and push image** | Push to `main` | Build Docker image, push to GitHub Container Registry. Image tag: `[version]-[branch]-[short sha]`. |

Deployment is **not** included. Use the image tag from the workflow run in your infra repo with ArgoCD.

---

## What your project needs

- **Dockerfile** at the repo root (and optionally `.dockerignore`).
- **`.nvmrc`** with the Node version (e.g. `20` or `22`).
- **`package.json`** with a `version` field (e.g. `"1.0.0"`) and scripts: `lint`, `typecheck`, `test`, `build`. Use no-op scripts if you don’t use one (e.g. `"typecheck": "echo ok"`).
- **GHCR:** The workflow uses `GITHUB_TOKEN`. Your infra cluster must be able to pull from GHCR (e.g. image pull secret or PAT).

---

## Option A – Reusable workflow (recommended)

No copy of the workflow logic. Your repo only defines triggers and calls the template.

**Requirements for reusable workflows:**

- The template repo **manusmd/github-workflows** must be **public**, so your app repo can access it. If it is private, reusable workflows only work when both repos are in the same GitHub organization (with a plan that allows it) and the template repo grants access.
- Use the **default branch** of the template repo in the `uses` line. If the default branch is `main`, use `@main`; if it is `master`, use `@master`. Reusable workflows are read from the default branch only.

### 1. Add CI workflow

Create `.github/workflows/ci.yml` in your app repo:

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
jobs:
  ci:
    uses: manusmd/github-workflows/.github/workflows/ci.yml@main
```

### 2. Add build-push workflow

Create `.github/workflows/build-push.yml` in your app repo:

```yaml
name: Build and push image
on:
  push:
    branches: [main]
jobs:
  build-push:
    uses: manusmd/github-workflows/.github/workflows/build-push.yml@main
    permissions:
      contents: read
      packages: write
```

The caller must grant `packages: write` so the reusable workflow can push the image to GHCR.

### 3. Pin a version (optional)

To avoid picking up template changes automatically, use a tag instead of `@main`:

```yaml
uses: manusmd/github-workflows/.github/workflows/ci.yml@v1
```

Create a tag (e.g. `v1`) in the template repo when you want to freeze the version.

---

## Option B – Copy the workflows

Copy the **contents** of these files from [manusmd/github-workflows](https://github.com/manusmd/github-workflows) into your app repo:

- `.github/workflows/ci.yml`
- `.github/workflows/build-push.yml`

You then own the YAML and can change it per repo. You won’t get updates from the template repo unless you copy again.

---

## Image tag format

Each push to `main` produces an image tagged as:

**`[version]-[branch]-[short sha]`**

Examples:

- `1.0.0-main-a1b2c3d`
- `2.1.3-main-f7e8d9c`

Full image: `ghcr.io/<owner>/<repo>:<tag>` (e.g. `ghcr.io/myorg/my-app:1.0.0-main-a1b2c3d`).

Branch names with `/` (e.g. `feature/foo`) become `-` in the tag (e.g. `1.0.0-feature-foo-a1b2c3d`).

---

## Getting the image tag for ArgoCD

1. In your app repo, go to **Actions**.
2. Open the latest **Build and push image** run.
3. In the run summary, find the **Image tag summary** step. It shows the full image and tag to use.
4. In your infrastructure repo, set that image (or tag) in ArgoCD / Helm so the cluster pulls the new version.

---

## Troubleshooting: "workflow was not found"

If you see `workflow was not found` when calling `manusmd/github-workflows/.github/workflows/ci.yml@main`:

1. **Make the template repo public**  
   In **manusmd/github-workflows**: Settings → General → Danger zone → Change visibility → Public. Then any repo can call the workflows.

2. **Use the template repo’s default branch**  
   In **manusmd/github-workflows**, check the default branch (Settings → General → Default branch). If it is `master`, use `@master` in the `uses` line instead of `@main`.

3. **If you keep the template repo private**  
   Your app repo must be in the **same organization** as manusmd/github-workflows, and the org must allow “Access to reusable workflows from private repositories”. In the template repo: Settings → Actions → General → “Access to reusable workflows from private repositories” and allow your org.

Until the template repo is public or the private-access setup is done, use **Option B – Copy the workflows** so the YAML lives in your app repo.

---

## Quick checklist

- [ ] Dockerfile at repo root
- [ ] `.nvmrc` with Node version
- [ ] `package.json` with `version` and `lint` / `typecheck` / `test` / `build` scripts
- [ ] `.github/workflows/ci.yml` and `.github/workflows/build-push.yml` added (reusable or copied)
- [ ] Push to `main` to run CI and build/push image
- [ ] Use the image tag from the Actions run in your infra repo for ArgoCD
