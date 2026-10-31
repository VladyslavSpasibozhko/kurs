# 01 — Repository and architecture

Status: ✅ approved · amended 2026-10-23 (`shared` split into `shared` + `http`, see section 02 and `00-decisions.md`); 2026-10-31 (plain CSS + CSS Modules instead of Tailwind; `repositories/` split into `tables/` + `multi/`, see section 09)

## Repository layout

```
kurs/
  package.json            pnpm workspace root: dev, lint, typecheck, build
  pnpm-workspace.yaml     server, client, shared, http
  biome.json              lint + format + import boundaries
  tsconfig.base.json      strict, shared settings
  docker-compose.yml      postgres only
  .env.example
  .github/workflows/ci.yml   PR title (Conventional Commits) → biome ci → typecheck → build

  shared/                 @kurs/shared (domain), only dependency is zod
    src/domain/           common, note, task, goal, day-plan
    src/inputs/           CreateXInput / UpdateXInput
    src/rules/            quadrantOf, isDoneOn, goalProgress
    src/errors.ts         error codes

  http/                   @kurs/http (HTTP contract), depends on @kurs/shared + zod
    src/define-route.ts   defineRoute, SuccessBody
    src/api-error.ts      ApiErrorSchema
    src/routes/           system, auth, notes, tasks, goals, day-plans

  server/                 Node 24 + Hono
    src/index.ts          start the HTTP server
    src/app.ts            Hono app: cors, logger, error handler, notFound, routes
    src/env.ts            .env validated with Zod at startup
    src/routes/           one file per resource; registered from @kurs/http via route(); requireAuth → userId
    src/services/         business logic (e.g. TaskService.setDone)
    src/repositories/     the only DB access; every query filters by userId; row → domain mapping
      tables/             one repository per table
      multi/              multi-table actions, one transaction each (uses tables/)
    src/auth/             password.ts (scrypt), tokens.ts (JWT + refresh), cookies.ts, auth.service.ts
    src/db/               schema.ts (Drizzle), client.ts, migrations/
    drizzle.config.ts

  client/                 React + Vite + TS
    src/main.tsx
    src/routes/           TanStack Router (file-based): login, register, _auth/account, _auth/notes, _auth/goals, _auth/day/$date
    src/api/              http.ts (request), *.queries.ts, query-keys.ts
    src/features/         page components: notes, goals, day
    src/i18n/             react-i18next, en.json
    src/styles/           global.css: CSS variables from the prototype + base styles
```

## Request flow

```
React component → useQuery → request(route, args) ──HTTP + cookie──▶ Hono route
   → zValidator (schemas from the @kurs/http route) → requireAuth (userId from JWT)
   → Service → Repository → Drizzle → Postgres
   ◀── JSON (shape = domain schema) ── on error: { code, message, details? }
```

## Layer rules

- Calls go only downward: `routes` → `services` → multi-table `repositories/multi` → table `repositories/tables` → `db`. Services may call either kind of repository; table repositories never call multi-table ones (section 09).
- Only `repositories` touch the database; only `repositories/multi` start transactions.
- `userId` comes only from the access token and is passed into every service and repository call.
- `shared` imports nothing from `http`, `server` or `client`; `http` imports nothing from `server` or `client`.
- `client` never imports from `server`; it talks to the server only through `@kurs/http`, `@kurs/shared` and HTTP.
- Server `services` use `@kurs/shared` only, never `@kurs/http`.

Enforced by:
- Between packages: pnpm workspace dependencies + TypeScript project references.
- Layers inside `server`: Biome `noRestrictedImports` with per-folder `overrides`.

## Running locally

```
docker compose up -d     # Postgres
pnpm db:migrate          # apply Drizzle migrations
pnpm dev                 # builds shared + http, then in parallel: tsc --watch for them, server (tsx watch, :3000), client (Vite, :5173)
```

- No Vite proxy. The client calls `VITE_API_URL` (`http://localhost:3000/api`) with `credentials: 'include'`.
- Different ports on `localhost` are the same site, so `SameSite` cookies work; the server enables CORS for `CLIENT_URL` with credentials.
- In production the client and API must share a registrable domain (e.g. `app.kurs.dev` + `api.kurs.dev`).
- Secrets live in `.env` (never committed); `.env.example` is committed. `JWT_SECRET` is server-only.
