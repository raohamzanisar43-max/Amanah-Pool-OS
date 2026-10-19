# CI/CD

| Workflow | Trigger | What it does |
|---|---|---|
| `ci.yml` | every pull request, push to `main`/`master` | Django `check`, migration drift check, test suite (Postgres 16), frontend lint + build, Docker image builds |
| `deploy.yml` | CI succeeded on a push to `main`/`master`, or manual run | rsync the repo to the server, `docker compose up -d --build`, verify `https://pool.novulabs.net/api/v1/health/` |

## Flow
`feature branch -> pull request (CI must pass) -> merge to master -> CI -> automatic deploy -> live`

## Production server
- Host: `/opt/pool` on the VPS; stack defined in `deploy/docker-compose.server.yml` (project name `pool`: db, backend, web).
- TLS and routing: host nginx (`deploy/nginx-pool.novulabs.net.conf`) + Let's Encrypt. `/api/` -> backend `:8040`, rest -> web `:3040` (loopback only).
- Secrets live only on the server in `/opt/pool/deploy/.env` (never in git; rsync skips it). Template: `deploy/.env.example`.
- GitHub secrets (Settings -> Secrets -> Actions, environment `production`): `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`.

## Operations
```bash
cd /opt/pool/deploy
docker compose -f docker-compose.server.yml ps
docker compose -f docker-compose.server.yml logs -f backend
docker compose -f docker-compose.server.yml exec db pg_dump -U amanah amanah > backup.sql
```
Rollback: revert the commit on `master`; the pipeline redeploys the previous state.
