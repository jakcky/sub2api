# Local Customizations and Update Runbook

This fork keeps local changes on top of upstream `Wei-Shaw/sub2api`. This file
is also the production update runbook for `/var/www/sub2api`.

## Instructions for AI Agents

When the user references this file and asks to update or sync upstream, perform
the workflow below end to end. Do not stop after fetching, merging, or giving
instructions unless a real blocker requires user input.

Important constraints:

- Preserve all customizations documented in this file.
- Treat existing uncommitted changes as user-owned; do not discard them.
- This server has limited memory. Do not build the application image locally by
  default.
- Build production images with GitHub Actions and pull them from GHCR.
- Never deploy an image until its GitHub Actions build has succeeded and the
  corresponding GHCR tag exists.
- Do not use `--no-cache` unless explicitly troubleshooting a cache or base-image
  problem.
- Do not prune the previous working image before the new container is healthy.

## Public GitHub Links

- Hide GitHub repository links from public or normal-user visible pages.
- Keep GitHub links that are only visible to administrators.
- Current public pages customized:
  - `frontend/src/views/HomeView.vue`
  - `frontend/src/views/KeyUsageView.vue`

Expected behavior:

- Public home footer should not show a GitHub link.
- Public API key usage footer should not show a GitHub link.
- Admin-only GitHub links, such as the admin header dropdown and admin settings
  help links, may remain.

## Production Image and Compose

Production images are built from this fork by
`.github/workflows/custom-image.yml` and published as:

```text
ghcr.io/jakcky/sub2api:custom-main
ghcr.io/jakcky/sub2api:custom-<full-git-sha>
```

`deploy/.env` must select the GHCR image:

```dotenv
SUB2API_IMAGE=ghcr.io/jakcky/sub2api:custom-main
```

Keep the local build configuration in `deploy/docker-compose.yml` as an
emergency fallback, even though normal production updates use `--no-build`:

```yaml
services:
  sub2api:
    image: ${SUB2API_IMAGE:-sub2api:custom}
    build:
      context: ..
      dockerfile: Dockerfile
```

## Upstream Update and Deployment Workflow

### 1. Inspect and merge upstream

Start from `/var/www/sub2api` and record the pre-merge commit:

```bash
cd /var/www/sub2api
git status --short --branch
BEFORE_UPDATE=$(git rev-parse HEAD)
git fetch --no-tags upstream main
git merge --no-edit upstream/main
git diff --name-status "$BEFORE_UPDATE"..HEAD
```

If there is a conflict, resolve it while preserving this file's requirements.
Do not discard unrelated user changes.

### 2. Verify local customizations

Before pushing:

- Confirm the public footers in `HomeView.vue` and `KeyUsageView.vue` contain no
  GitHub repository link.
- Confirm `deploy/docker-compose.yml` still contains both the customizable
  `image:` value and the local `build:` fallback shown above.
- Run `docker compose -f deploy/docker-compose.yml config --quiet`.
- Review `git status` and the merge diff.

### 3. Push the fork

Push only after verification succeeds:

```bash
git push origin main
```

The expected remotes are:

```text
origin   git@github.com:jakcky/sub2api.git
upstream https://github.com/Wei-Shaw/sub2api.git
```

### 4. Wait for the cloud image

Pushing `main` automatically runs `Build Custom Image` when any of these paths
change:

```text
Dockerfile
frontend/**
backend/**
docs/legal/**
deploy/docker-entrypoint.sh
.github/workflows/custom-image.yml
```

Wait for the workflow to succeed. Then verify the immutable image tag from the
server:

```bash
HEAD_SHA=$(git rev-parse HEAD)
docker manifest inspect "ghcr.io/jakcky/sub2api:custom-${HEAD_SHA}"
```

If the manifest does not exist or the workflow failed, stop and diagnose the
Actions failure. Do not replace the running container.

If only files outside the trigger list changed, such as README files or sponsor
assets, no runtime image is needed. Push the merge, skip deployment, and report
that the running service was intentionally left unchanged.

### 5. Pull and deploy without building

For a runtime update whose Actions build succeeded:

```bash
cd /var/www/sub2api/deploy
docker compose pull sub2api
docker compose up -d --no-build sub2api
docker compose ps
curl -fsS http://127.0.0.1:8080/health
```

Verify that:

- The `sub2api` container uses `ghcr.io/jakcky/sub2api:custom-main`.
- The application, PostgreSQL, and Redis containers are healthy.
- `/health` returns a successful response.
- Memory remains stable after the replacement.

If the new container is unhealthy, inspect its logs and restore the previous
working image. Do not rebuild locally as the first recovery attempt.

## Manual Actions Build

The workflow can be run manually from GitHub:

```text
Actions -> Build Custom Image -> Run workflow -> main
```

Use this only when a runtime image is required but the path-filtered push did
not trigger a build. Normal upstream updates should rely on the automatic push
trigger.

## Local Build Exception

Run a local Docker build only when the user explicitly requests it or GHCR is
unavailable and deployment cannot wait. Check available memory and swap first,
avoid `--no-cache`, and use a resource-limited Buildx builder if one is
available. A memory limit prevents the host from being exhausted but may cause
the build to fail; it does not reduce the application's actual build needs.
