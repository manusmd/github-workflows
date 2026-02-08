# Pipeline usage guide

How to use the [manusmd/github-workflows](https://github.com/manusmd/github-workflows) pipeline in a project (e.g. Next.js app). One workflow runs CI and optionally builds/pushes a Docker image to GHCR. Deployment is done manually via ArgoCD in your infrastructure repo.

---

## What the pipeline does

One workflow with two jobs:

| Job | When it runs | Purpose |
|-----|--------------|---------|
| **ci** | Always | Install deps, dependency audit (fails on high/critical), lint, typecheck, test, Next.js build. Node version from `.nvmrc`. |
| **build-push** | Only when `push_image: true` (e.g. push to `main`) | Build Docker image, push to GHCR. Image tag: `[version]-[branch]-[short sha]`. |

On **pull requests** to `main`: only CI runs. On **push** to `main`: CI runs, then build-push runs. Deployment is **not** included; use the image tag in your infra repo with ArgoCD.

---

## What your project needs

- **Dockerfile** at the repo root (and optionally `.dockerignore`).
- **`.nvmrc`** with the Node version (e.g. `20` or `22`).
- **`package.json`** with a `version` field (e.g. `"1.0.0"`) and scripts: `lint`, `typecheck`, `test`, `build`. Use no-op scripts if you don’t use one (e.g. `"typecheck": "echo ok"`).
- **GHCR:** The workflow uses `GITHUB_TOKEN`. Your infra cluster must be able to pull from GHCR (e.g. image pull secret or PAT).

---

## Option A – Reusable workflow (recommended)

One file in your app repo. No copy of the pipeline logic.

**Requirements:**

- The template repo **manusmd/github-workflows** must be **public** (or both repos in the same org with access to reusable workflows from private repos).
- Use the template repo’s **default branch** in the `uses` line (`@main` or `@master`).

### Add the pipeline

Create `.github/workflows/pipeline.yml` in your app repo:

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
    uses: manusmd/github-workflows/.github/workflows/pipeline.yml@main
    with:
      push_image: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
```

- **Push or PR to `main`** → CI runs.
- **Push to `main`** → CI runs, then image is built and pushed. PRs do not push an image.

To pin a version, use a tag (e.g. `@v1`) instead of `@main` in the `uses` line.

---

## Option B – Copy the workflow

Copy the **contents** of [manusmd/github-workflows/.github/workflows/pipeline.yml](https://github.com/manusmd/github-workflows/blob/main/.github/workflows/pipeline.yml) into your app repo as `.github/workflows/pipeline.yml`. You then need to add triggers and pass the input yourself, e.g.:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
# ... then the jobs from the template, with push_image set as in Option A
```

Or copy the file and add the `on` and `with` at the top. You own the YAML and won’t get updates from the template repo unless you copy again.

---

## Image tag format

When the image is pushed (push to `main`), the tag is:

**`[version]-[branch]-[short sha]`**

Examples: `1.0.0-main-a1b2c3d`, `2.1.3-main-f7e8d9c`.

Full image: `ghcr.io/<owner>/<repo>:<tag>`. Branch names with `/` become `-` in the tag.

---

## Getting the image tag for ArgoCD

1. In your app repo, go to **Actions**.
2. Open the latest **Pipeline** run (from a push to `main`).
3. In the run summary, open the **build-push** job and find the **Image tag summary** step. It shows the full image and tag.
4. Use that image (or tag) in your infrastructure repo (ArgoCD / Helm).

---

## Troubleshooting: "workflow was not found"

If you see `workflow was not found` when calling `manusmd/github-workflows/.github/workflows/pipeline.yml@main`:

1. **Make the template repo public**  
   In **manusmd/github-workflows**: Settings → General → Danger zone → Change visibility → Public.

2. **Use the template repo’s default branch**  
   If the default branch is `master`, use `@master` instead of `@main`.

3. **If the template repo stays private**  
   Your app repo must be in the **same organization** and the org must allow “Access to reusable workflows from private repositories” for that repo.

Until then, use **Option B – Copy** so the workflow lives in your app repo.

---

## Troubleshooting: "packages: write" / permissions

If the build-push job fails with permission errors, ensure your **caller** workflow has at **workflow** level:

```yaml
permissions:
  contents: read
  packages: write
```

(same indentation as `name` and `on`).

---

## Quick checklist

- [ ] Dockerfile at repo root
- [ ] `.nvmrc` with Node version
- [ ] `package.json` with `version` and `lint` / `typecheck` / `test` / `build` scripts
- [ ] `.github/workflows/pipeline.yml` added (reusable or copied)
- [ ] Push to `main` to run CI and build/push image
- [ ] Use the image tag from the Pipeline run in your infra repo for ArgoCD
