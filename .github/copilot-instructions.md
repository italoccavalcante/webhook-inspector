## Quick orientation

This repo is a small monorepo with two TypeScript workspaces: `api/` (Fastify + Drizzle/Postgres) and `web/` (Vite + React + TanStack Router).

- API entry: `api/src/server.ts` (Fastify server, OpenAPI at `/docs`).
- Database schema & migrations: `api/src/db/schema/**`, migrations in `api/src/db/migrations` and `api/drizzle.config.ts`.
- Capture route (core feature): `api/src/routes/capture-webhook.ts` — all incoming webhooks are saved to the `webhooks` table defined in `api/src/db/schema/webhooks.ts`.
- Frontend entry: `web/src/main.tsx`. Routes live under `web/src/routes` and a generated router tree lives at `web/src/routeTree.gen.ts`.

## What I need to know to be productive

- Environment: the API expects a `DATABASE_URL` env var. The API dev script loads `.env` (`api` uses `tsx --env-file=.env src/server.ts`). Default HTTP port is `3333` (see `api/src/env.ts`).
- DB: PostgreSQL via `pg` + `drizzle-orm`. Migrations are managed with `drizzle-kit` (`api` scripts: `db:generate`, `db:migrate`, `db:studio`). The schema is exported from `api/src/db/schema/index.ts` and used by `drizzle.config.ts`.
- TypeScript path alias: in the API, imports use `@/…` mapped to `api/src/*` (see `api/tsconfig.json`). Prefer the same alias when editing API source.
- API validation & types: Fastify + `fastify-type-provider-zod`. Route schemas are Zod-based. Response/request shapes are defined inline in route files.

## Useful commands

Run API in dev (loads `.env` at `api/.env`):

  cd api
  npm install
  npm run dev

Run frontend in dev:

  cd web
  npm install
  npm run dev

Build frontend:

  cd web
  npm run build

Run DB migrations and studio (from `api`):

  cd api
  npm run db:migrate
  npm run db:studio

Formatting (both workspaces use Biome):

  cd api && npm run format
  cd web && npm run format

Notes: the repo uses workspaces (root `package.json`) — you can also run `npm --workspace=api run dev` from the root with npm 9+.

## Conventions & patterns specific to this repo

- Webhook capture: the route `app.all('/capture/*', ...)` (see `api/src/routes/capture-webhook.ts`) strips `/capture` then stores the remainder as `pathname`. The handler normalizes headers into a simple object and stores the raw body (or JSON-stringified body) in `body`.
- DB shape: `webhooks` table uses text `id` generated with `uuidv7()` and stores headers as `jsonb` and `body` as a string (`api/src/db/schema/webhooks.ts`). Expect `statusCode`, `method`, `pathname`, `ip`, `contentType`, `contentLength` fields.
- Absolute imports: API code uses `@/` alias. When adding new files, prefer `@/` imports to stay consistent with existing files.
- Routes register as Fastify plugins in `api/src/server.ts` — add new endpoints by creating a route file in `api/src/routes` and registering it in `server.ts`.

## Integration points

- Capture endpoint: external systems should POST to `http://<host>:3333/capture/<your-path>` with any method; the response is `201` with `{ id }` (see capture route).
- Database: `env.DATABASE_URL` must point to a Postgres DB. `drizzle-kit` migrations write to `api/src/db/migrations`.
- API docs: OpenAPI/Swagger is available at `/docs` (Fastify + Scalar API reference plugin).

## Small examples

- Inspect a saved webhook (example):
  - POST a JSON payload to `http://localhost:3333/capture/orders/created`
  - The API returns `{ id: "<uuid>" }` and a row is inserted into `webhooks`.

- Add a route: create `api/src/routes/new-route.ts`, export a Fastify plugin, then add `app.register(newRoute)` in `api/src/server.ts`.

## Where to look for more context

- API source: `api/src/*` (routes: `api/src/routes`, db: `api/src/db`)
- Frontend source: `web/src/*` (routes: `web/src/routes`, components: `web/src/components`)
- Build & scripts: `api/package.json`, `web/package.json` (see scripts for dev/build/migrate/tools used)

If anything here is incomplete or you want more examples (e.g., sample `.env`, a minimal local Postgres docker-compose snippet, or test guidance), tell me which area to expand and I'll update this file.
