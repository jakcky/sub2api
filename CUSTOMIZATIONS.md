# Local Customizations

This fork keeps a small set of local changes on top of upstream
`Wei-Shaw/sub2api`. When merging upstream updates, preserve the requirements
below.

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

## Docker Compose Build

The production compose file should build the local fork instead of always using
the upstream image.

Keep this shape in `deploy/docker-compose.yml`:

```yaml
services:
  sub2api:
    image: ${SUB2API_IMAGE:-sub2api:custom}
    build:
      context: ..
      dockerfile: Dockerfile
```

Server build commands:

```bash
cd /var/www/sub2api
git pull origin main
cd deploy
docker compose build sub2api
docker compose up -d sub2api
```

Use `--no-cache` only when troubleshooting dependency or base-image issues.

## Upstream Merge Workflow

Recommended local workflow:

```bash
git fetch origin
git merge origin/main
```

After resolving conflicts, verify the customizations above are still present,
then push to the fork:

```bash
git push jakcky main
```

On the server, `origin` should point to:

```text
https://github.com/jakcky/sub2api.git
```
