# Self-hosting Autonoma on Railway

Deploys the Autonoma platform (api + ui + Temporal + Postgres + Redis) to a **dedicated Railway project** so generated artifacts stay on infrastructure you control. Fork: `NesberryEnterprise/autonoma`.

> **Secrets policy:** never commit real secret values to this repo. All secrets
> live ONLY as Railway service variables. Placeholders below are intentional.

> Status: build Dockerfiles authored (`apps/api/Dockerfile.railway`, `apps/ui/Dockerfile.railway`) — expect 1–2 iteration cycles on the first deploy (pnpm monorepo Docker builds usually need a tweak). Requires an active Railway paid plan (5 always-on services exceed the trial credit).

## Services (create in this order)

| # | Service | Source | Notes |
|---|---------|--------|-------|
| 1 | **Postgres** | Railway Postgres plugin | provides `DATABASE_URL`; Temporal also uses this DB |
| 2 | **Redis** | Railway Redis plugin | provides `REDIS_URL` |
| 3 | **temporal** | Docker image `temporalio/auto-setup:1.25.2` | env: `DB=postgres12`, `DB_PORT=5432`, `POSTGRES_USER`, `POSTGRES_PWD`, `POSTGRES_SEEDS=<postgres private host>`. Exposes 7233. |
| 4 | **api** (the `autonoma` service) | repo, Dockerfile via `RAILWAY_DOCKERFILE_PATH=apps/api/Dockerfile.railway` | port 4000; run `pnpm db:migrate` once after first deploy |
| 5 | **ui** | repo, Dockerfile via `RAILWAY_DOCKERFILE_PATH=apps/ui/Dockerfile.railway` | nginx on 3000; set **build** var `VITE_API_URL` = api public URL |

## Environment variables (api service) — set in Railway, NOT here

```
NODE_ENV=production
DATABASE_URL=${{Postgres.DATABASE_URL}}
REDIS_URL=${{Redis.REDIS_URL}}
API_PORT=4000
APP_URL=https://<ui-public-url>
ALLOWED_ORIGINS=https://<ui-public-url>
BETTER_AUTH_SECRET=<random 32-byte hex — generate, store in Railway only>
BETTER_AUTH_URL=https://<api-public-url>
SCENARIO_ENCRYPTION_KEY=<random 32-byte hex — generate, store in Railway only>
STRIPE_ENABLED=false
GOOGLE_CLIENT_ID=<from Google Cloud console>
GOOGLE_CLIENT_SECRET=<from Google Cloud console>
TEMPORAL_ADDRESS=${{temporal.RAILWAY_PRIVATE_DOMAIN}}:7233
TEMPORAL_NAMESPACE=default
INTERNAL_DOMAIN=autonoma.app
# AI keys only needed to RUN tests (not for the upload/dashboard); wire Ollama later.
```

UI service needs **build** var: `VITE_API_URL=https://<api-public-url>` (Vite inlines it at build time).

## Google OAuth (login is Google-only — no email/password path in `apps/api/src/auth.ts`)

1. Google Cloud console → APIs & Services → Credentials → OAuth client (Web).
2. Authorized redirect URI: `https://<api-public-url>/api/auth/callback/google`.
3. Put the client id/secret into the api env vars above.

## After deploy

1. Open the **ui** URL → sign in with Google → an organization is auto-created.
2. Generate an **API key** in the dashboard (Better-Auth `apiKey` plugin → `ask_...`).
3. Point the planner CLI at your instance and re-run to upload the existing artifacts:
   ```
   AUTONOMA_API_URL=https://<api-public-url>
   AUTONOMA_API_TOKEN=ask_<your new key>
   AUTONOMA_GENERATION_ID=<your generation id>
   npx @autonoma-ai/planner@latest --resume --non-interactive
   ```

## Known iteration risks (first deploy)

- `pnpm --filter ... deploy --prod /out` flags can differ by pnpm version — if the api build fails at the deploy step, fall back to copying `apps/api/dist` + a pruned `node_modules`.
- Temporal `auto-setup` needs Postgres reachable on the private network before it boots.
- The ui must be **rebuilt** whenever the api URL changes (Vite build-time inlining).
