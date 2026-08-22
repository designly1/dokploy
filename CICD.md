# CI/CD (designly1 fork)

This fork uses a split-branch workflow so fork-only CI/CD never ends up in pull requests to [Dokploy/dokploy](https://github.com/Dokploy/dokploy).

## Branch structure

| Branch | Purpose | Tracks |
|--------|---------|--------|
| **`canary`** | Sync with upstream Dokploy — no CI/CD, no feature work | `upstream/canary` |
| **`ci`** | Fork-only: GHCR builds + upstream workflow guards | `origin/ci` |
| **`feature/*`** | PR-ready feature work | `origin/feature/*` |

## Registry

Images are published to GitHub Container Registry:

```
ghcr.io/designly1/dokploy
```

Tags pushed from the **`ci`** branch:

| Tag | When |
|-----|------|
| `latest` | Every push to `ci` |
| `ci` | Every push to `ci` |
| `<version>` | Every push to `ci` (from `apps/dokploy/package.json`) |
| `sha-<commit>` | Every push to `ci` |

Workflow file: [`.github/workflows/ghcr.yml`](.github/workflows/ghcr.yml)

Auth uses the built-in `GITHUB_TOKEN` — no extra secrets are required for pushes from this repo.

## Day-to-day workflow

### Sync upstream

```bash
git checkout canary
git pull upstream canary
git push origin canary
```

### Build and deploy a custom image

Merge upstream into `ci`, push, and let GitHub Actions build:

```bash
git checkout ci
git merge canary
git push origin ci
```

Watch the run: [Actions → Build and Push to GHCR](https://github.com/designly1/dokploy/actions/workflows/ghcr.yml)

Deploy on the server after the build succeeds:

```bash
# Log in once if the package is private
echo "$GITHUB_TOKEN" | docker login ghcr.io -u designly1 --password-stdin

docker pull ghcr.io/designly1/dokploy:latest

docker service update \
  --image ghcr.io/designly1/dokploy:latest \
  --force \
  dokploy
```

### Work on a feature

Branch from synced `canary`, not from `ci`:

```bash
git checkout canary
git pull upstream canary

git checkout -b feature/my-thing
# ... make changes ...
git push -u origin feature/my-thing
```

### Open a pull request to upstream

```bash
gh pr create \
  --repo Dokploy/dokploy \
  --base canary \
  --head designly1:feature/my-thing
```

PR branches should contain **feature commits only**. They must not include:

- `.github/workflows/ghcr.yml`
- Fork guards added to upstream workflows (`dokploy.yml`, `deploy.yml`, etc.)

Those files live exclusively on the **`ci`** branch.

## Why upstream workflows fail on this fork

The upstream repo ships workflows that push to Docker Hub (`dokploy/dokploy`). This fork does not have `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets, so those workflows are guarded to skip when `github.repository != 'Dokploy/dokploy'`.

Only **Build and Push to GHCR** should run on pushes to **`ci`**.

## Remotes

```bash
origin    https://github.com/designly1/dokploy.git   # your fork
upstream  https://github.com/Dokploy/dokploy.git     # upstream
```

If `upstream` is missing:

```bash
git remote add upstream https://github.com/Dokploy/dokploy.git
git fetch upstream
```

## Current feature branch

Compose registry work lives on **`feature/compose-registry`**. Open PRs from that branch (or future `feature/*` branches), not from `ci` or `canary`.
