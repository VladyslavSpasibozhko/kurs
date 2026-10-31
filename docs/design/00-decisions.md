# 00 — Decisions log

Decisions made while planning, with the reason behind each. Newest topics at the bottom.

## Goal

- The project is both a learning project and a tool for daily use. Build properly, but every stage must end with something usable.

## Stack

| Area | Decision | Why |
|---|---|---|
| Repo | One repo, four folders: `server/`, `client/`, `shared/`, `http/` (pnpm workspaces) | One PR changes contract, server and client together |
| Language | TypeScript everywhere | Shared types via Zod |
| Server | Node 24 + Hono | Modern, minimal, little magic; popular (~45M weekly downloads, Oct 2026) |
| Contract | Domain Zod schemas in `@kurs/shared`; HTTP routes in `@kurs/http`; types via `z.infer` | One source of truth for validation, client types and (later) MCP tool schemas |
| OpenAPI / Swagger | Not used | Shared Zod package replaces generated client |
| Rejected: ts-rest | Last stable release Mar 2025; Hono adapter abandoned (Nov 2023) | |
| Rejected: Hono RPC | Couples client to server code at compile time | |
| Postgres access | Drizzle ORM + drizzle-kit migrations | Schema, types and migrations from one place; close to SQL |
| MongoDB | Not now — added with reflections | Mongo was only used for reflections, which are postponed |
| Client | React + Vite + TypeScript | |
| Client HTTP | Own `request(route, { params, query, body })` typed from `@kurs/http` | Explicit, no generation |
| Client data | TanStack Query on top of `request()` | Caching, loading/error states, invalidation after mutations |
| Routing | TanStack Router (file-based) | Typed params and search params with Zod |
| Styling | Plain CSS + CSS Modules; prototype colours/sizes as CSS variables in `global.css`; native elements (`<dialog>`, `<input type="date">`, `<select>`) | The prototype is already plain CSS, so styles port almost as is; nothing to install; no component library |
| i18n | react-i18next set up now; English only; English is default | Adding a language later = adding a file |
| Lint/format | Biome, with import boundaries via `noRestrictedImports` + overrides | One fast tool |
| Tests | None in the first stages | Manual checklist per stage until tests are added |
| Local env | Docker Compose with Postgres only; server and client run via `pnpm dev` | Faster reloads, simpler debugging |
| Vite proxy | Not used; client and API on different ports, same site | CORS with credentials on the server |

## Shared packages (decided 2026-10-23, see section 02)

- Two packages: `@kurs/shared` (domain: entities, inputs, pure rules, error codes) and `@kurs/http` (HTTP contract: `defineRoute`, routes, `ApiErrorSchema`).
  Why: the domain stays free of HTTP; services and future MCP tools depend only on `@kurs/shared`; package boundaries are enforced by pnpm dependencies instead of lint rules.
- Inputs (`CreateXInput`, `UpdateXInput`) live in `@kurs/shared/inputs` (they are what services and MCP tools accept), not in the HTTP contract.
- Route contract kept (not "schemas only"): the server registers its routes from it via `route()`, the client calls through `request()`. Why: path, method and response type are checked at compile time on both sides, and the server cannot drift from the contract.
- One success `response` per route, success status defaults by method (GET/PATCH/PUT 200, POST 201, DELETE 204), errors are always the common `ApiErrorSchema` and are not listed per route. Why: no OpenAPI consumes a per-route error list, so listing errors would add typing work for no payoff.
- Route `params` are written explicitly, not inferred from the path string.
- No runtime validation of responses on the client.
- Boundaries: between packages via pnpm dependencies + TS project references; layers inside `server` via Biome.
- DB schemas stay in `server/src/db`, never in `shared` or `http`.
- Build: `shared` and `http` each build themselves (`tsc -b` → `dist/` with JS, `.d.ts`, declaration maps); `exports` point at `dist/`; server and client consume them as regular packages. Why: each package is responsible for building and exposing its code. `pnpm dev` runs `tsc -b --watch` for both so `dist/` stays fresh. Alternative rejected: exporting `.ts` sources and letting tsx/Vite compile them (no build, but consumers depend on how the package is written).
- Server production build: plain `tsc` (no bundler).

## Authentication

**Email + password first (2026-10-31, approved — replaces the earlier Google-only decision; Google is added later):**
- Open sign-up: anyone can register. Why: data is already isolated per `userId`; a sign-up switch (`SIGNUP_ENABLED`) can be added when the app is deployed publicly.
- Register logs the user in: `POST /auth/register` sets the session cookies like login. Why: no email confirmation step exists, so a second login adds nothing.
- Email normalized in `@kurs/shared`: `EmailSchema = z.string().trim().toLowerCase().pipe(z.email())`, used by register and login; DB keeps plain `UNIQUE(email)`. No provider-specific rules (Gmail dots, `+tags`). Why: one rule in one place; rejected `citext` / `UNIQUE(lower(email))` (every lookup must remember it).
- Passwords hashed with `scrypt` from `node:crypto` (no dependency), in `server/src/auth/password.ts` (`hashPassword` / `verifyPassword`). Params N=2^16, r=8, p=2 (OWASP option, ~64 MiB), 16-byte random salt, 64-byte key. Stored as one string in `users.passwordHash`: `scrypt$N=65536,r=8,p=2$<salt b64>$<hash b64>`; verify parses params from the string and compares with `timingSafeEqual`. Rejected: argon2id via `@node-rs/argon2` (extra native dependency), bcrypt (72-byte limit).
- scrypt settings live in one code constant `SCRYPT_CONFIG = { keyLen: 64, saltLen: 16, N: 2**16, r: 8, p: 2, maxmem: 128 MiB }` (server config, not `.env`). `hashPassword` uses it for new hashes; `verifyPassword` reads N/r/p/salt from the stored string and takes only `maxmem` from the config. Why: changing hashing strength is a code change, not a per-environment setting.
- Password rules: `PasswordSchema = z.string().min(8).max(128)` in `@kurs/shared` (register + client form). No character-mix rules, no common-password blocklist, no strength meter, no trimming or normalization (hash exactly what was typed). Login checks only `z.string().min(1)`, so tightening the rule never locks out existing users. Why: NIST SP 800-63B — length matters, composition rules don't; max 128 bounds hashing cost. Possible later: small common-password list, NFKC normalization.
- No email verification for now (planned later, with password reset — same email service and token table). Rule for Google later: never auto-link a Google account to an existing account by matching email (the email was never verified).
- No password reset in the app for now (planned later, by email link, together with email verification). No stand-in until then (decided 2026-10-31: the `set-password` admin script was dropped); a forgotten password cannot be recovered in the app.
- Change password now: 🔒 `PATCH /auth/password { currentPassword, newPassword }` → 204. Wrong `currentPassword` → `INVALID_CREDENTIALS`; `newPassword` uses `PasswordSchema`. Sessions are not touched: the current device and other devices stay logged in (user's choice: revoking other devices is not needed). Needs a small account/settings page in the client.
- "Log out everywhere" not planned.
- Login failure: unknown email and wrong password both → `401 INVALID_CREDENTIALS` ("Wrong email or password"). Unknown email still runs one `verifyPassword` against a fixed dummy hash, so timing doesn't reveal registered emails. `INVALID_CREDENTIALS` added to `ErrorCode` (amends section 02).
- Register with a taken email → `409 CONFLICT` (reuses the existing code; the register form translates it to "email already registered"). Insert directly and map the Postgres unique violation `23505` to `CONFLICT` (no check-then-insert race). Reveals that the email exists — accepted; hiding it needs the email service.
- No rate limiting for now (known gap; must be done before public deployment): planned in-memory limit on failed logins per IP + email (e.g. 5 per 15 min → `429`), looser per-IP limit on register. Rejected: account lockout (lets anyone lock a user out).
- Auth routes, all JSON in `@kurs/http`: `POST /auth/register` (201 `{ userId }` + cookies), `POST /auth/login` (200 `{ userId }` + cookies), `POST /auth/refresh` (200 `{ userId }` + new cookies), `POST /auth/logout` (204, not 🔒, always succeeds), 🔒 `GET /auth/session` (200 `{ userId }`), 🔒 `PATCH /auth/password` (204). Inputs `RegisterInput`, `LoginInput`, `ChangePasswordInput` in `@kurs/shared/inputs`. Responses carry only `userId` (no profile in the UI).
- Sessions: access JWT + refresh token with rotation and reuse detection (unchanged). Parallel refresh is fixed on the client only: `request()` keeps one shared in-flight refresh promise, every 401 waits for it. `refresh_tokens.used` stays a boolean (no grace window). Accepted: two tabs refreshing at the same moment can trigger reuse detection and log the user out. Possible later: Web Locks (`navigator.locks`) so only one tab refreshes.
- Cookies: `access` (JWT, `Path=/api`) and `refresh` (32 random bytes, DB keeps SHA-256, `Path=/api/auth`); both HttpOnly, SameSite=Lax, Secure in prod. JWT HS256 via `hono/jwt`, `JWT_SECRET` (≥ 32 chars) in the `.env` schema. Logout marks the refresh token used, clears cookies, always 204. No CSRF tokens (SameSite=Lax + CORS preflight). `requireAuth` checks only the JWT, no DB lookup.
- `GET /auth/session` for the startup session check (no user profile needed in the UI).

**Google login (later):** `openid-client` v6 (Authorization Code + PKCE; `arctic` rejected — deprecated), state + verifier in a short-lived HttpOnly cookie. Migration adds `users.googleId` (nullable, unique) and makes `passwordHash` nullable.
- Token lifetimes: access JWT 15 min, refresh token 30 days sliding (each refresh issues a new 30-day token), no absolute limit. Code constants `ACCESS_TTL`, `REFRESH_TTL` in server config (not `.env`).
- Refresh tokens are never deleted (no cleanup job, no delete on login/refresh). Why: simplest; growth (~3k rows/month per active device) is harmless at this scale. Used rows must stay anyway for reuse detection until they expire.

## Data model

- Task ↔ goal via join table `task_in_goal` with `UNIQUE(taskId)`; `weight` renamed to `priority`.
- `task_for_day.finished` removed (decided 2026-10-23; earlier kept as a column). "Done on day D" = `tasks.doneAt === D`. Why: nothing to keep in sync and no question of which day a reopen touches. Cost: history follows current state — a task done, reopened and done on a later day no longer shows done on the first day.
- `important` / `urgent` flags stay on tasks even though the matrix page is postponed (quadrant is still shown per task and counted in Progress).
- «Терміново» is never set automatically (per «Прийняті рішення» in the requirements).
- A goal is completed only manually.
- No system "Without goal" goal (decided 2026-10-23). "No goal" = no `task_in_goal` row. Why: a system goal would need creating at signup and excluding from active-goal counts, progress, the day picker's goal steps and AI `list_goals`, plus guards against deleting or finishing it — more special cases than a null check.
- Goals page: two parallel requests, `GET /goals/tasks` + `GET /tasks?goal=none`, both with a done/todo filter (default `todo`). Why: separate cache keys, a mutation refetches only what it touches. Rejected: one `GET /goals/overview`.
- Domain `Task` is plain (changed 2026-10-23; earlier it carried `goalId` / `priority`). Task endpoints return `TaskWithGoal { ...task, priority, goal: Goal | null }` (second pass; earlier nested `TaskItem`), filled by the repository via `LEFT JOIN task_in_goal`; the client never sees the join table.
- Dates (decided 2026-10-23): days (`day_plans.day`, `dueDate`, `doneAt`) are `YYYY-MM-DD` strings / Postgres `date`; moments (`createdAt`, `updatedAt`, `completedAt`, `expiresAt`) are ISO strings / `timestamptz` in UTC. Why: a day stored as a timestamp moves to another day when read in another time zone or across DST. Rejected: epoch-ms numbers; UTC timestamps for days with client-computed ranges (a plan made in one zone is not found from another). Time zones and DST are not a concern for now.
- "Today" is decided by the client (decided 2026-10-23): the server does not know the time zone and does not enforce past-day read-only; that is a UI rule. `PAST_DAY_READONLY` error code and `editable` field dropped. Why: simplest option; rejected `X-Timezone` header and `users.timezone`.
- Deleting a goal (decided 2026-10-23): two modes, chosen by the user — `keep` (goal and `task_in_goal` rows removed, tasks stay without a goal) and `delete` (goal, `task_in_goal` rows and the tasks removed). Why: the user wants both a safe option and a full cleanup. `delete` also removes those tasks from past-day history; the confirmation dialog warns about it. Not in the source docs (they have no goal deletion).

## Day plan (decided 2026-10-23)

- No `candidates` endpoint and no combined "add tasks + convert notes" body. Why: each endpoint does one job; the picker reuses existing lists (`/tasks?plannedBefore=`, `/goals`, `/tasks?goal=none`, `/notes`) and shares their cache.
- `POST /day-plans/:date/items { taskIds }` only adds plan rows; a list (not one request per task) so a selection is saved all or nothing.
- A note is added to a day individually via `POST /notes/:id/convert` with `{ type: 'task', input: { …, planDate } }`.
- A goal step in the day plan carries its goal (`goal: { id, title }`).
- Day plan grouped by goal, not by quadrant (decided 2026-10-23; grouping moved to the client in the second pass, see below): originally `GET /day-plans/:date` → `{ date, groups: [{ goal: { id, title }, tasks: TaskItem[] }, …, { goal: null, tasks }] }`, «Без цілі» last; only non-empty groups. «+ Додати» sits in each goal group and creates the task with that `goalId`. Why: the user wants the day organised by goal. Rejected: quadrant → goal nesting, goal → quadrant nesting, client-side quadrant split.

## Notes (decided 2026-10-23)

- One endpoint `POST /notes/:id/convert` replaces `to-task` and `to-goal`. Body `{ type: 'task', input: CreateTaskInput } | { type: 'goal', input: CreateGoalInput }`, response `{ type: 'task', task: TaskWithGoal } | { type: 'goal', goal: GoalWithTasks }` (201). Why: one endpoint per note action; another conversion target later is one more union member.
- Inputs nested under `input`, not flattened next to `type`. Why: reuses `CreateTaskInput` / `CreateGoalInput` unchanged; flattening would need new merged schemas with fields (`steps`, `planDate`) that only fit one type.
- Still one success `response` per route (section 02): the response is one schema that is a union. Services stay separate (`NoteService.convertToTask` / `convertToGoal`) so MCP tools can call them directly.

## Tasks (decided 2026-10-23)

- One `PATCH /tasks/:id/status` with `{ status: 'done', date } | { status: 'todo' }` replaces `POST /tasks/:id/complete` and `/reopen`. Returns `Task` (200). Why: one endpoint for status; a discriminated union makes `date` required only for `done`. Kept apart from `PATCH /tasks/:id` because of that required `date`.
- ~~`TaskItem { task, goal: { id, title } | null, priority }` is the one shape~~ (replaced by `TaskWithGoal` in the second pass, see below) for a task with its goal: `GET /tasks`, `GET /tasks/by-goal`, day-plan items, and the responses of `POST /tasks`, `PATCH /tasks/:id`, `PATCH /tasks/:id/status`. Why: the UI shows the goal title next to a task, and mutation responses carry what the cache needs. Rejected: `goalId` on `Task` (client would look the goal up separately).
- ~~New `GET /tasks/by-goal`~~ (dropped in the second pass, see below) → `[{ goal: Goal, tasks: TaskItem[] }, …, { goal: null, tasks }]` («Без цілі» last). Added alongside `GET /goals/tasks` (goals with their steps) and `GET /tasks?goal=none`, which stay as they are for now.
- `GET /tasks?goal=<id>` dropped; only `goal=none` remains. Why: a goal's steps come from `GET /goals/tasks`.

## Goals API (decided 2026-10-23)

- `GET /goals?status&stepStatus` removed; replaced by `GET /goals/tasks?status=active|done&taskStatus=todo|done`. Both params optional; omitted = no filter (the old `all` value and `todo` default are gone; the Goals page sends `status=active&taskStatus=todo`). Goals sorted by `createdAt`.
- Shape: `GoalWithTasks = Goal & { tasks: StepItem[] }`, `StepItem = Task & { priority }` — flat, no nested `task` or `goal` (the parent is the goal). `TaskWithGoal` is used everywhere else.
- `POST /goals` (201), `PATCH /goals/:id` (200) and `POST /notes/:id/convert` with `type: 'goal'` return this same shape.
- Shapes for a task, by context: `Task` (plain), `StepItem` inside a goal, `TaskWithGoal` everywhere else.
- `GET /goals?status=active|done` brought back as a light list: `Goal[]` with no tasks, `status` optional. For the task form's goal dropdown.
- Goal status changes via `PATCH /goals/:id/status { status: 'done' | 'active' }`, same convention as tasks; `UpdateGoalInput` has no `status`. `done` sets `completedAt`; `done → active` (reopen) is allowed and clears `completedAt`.
- Goal `progress` is postponed: not on `Goal` for now; decided with the Progress section.
- Progress page postponed (2026-10-31): implementation starts without it; no `/progress/*` routes, no `ProgressSummary` / `QuadrantCounts`. Why: not needed to start; it can be built later from existing lists.
- Delete mode names: `keep` / `delete` everywhere (query param, service, docs); `keep-steps` / `delete-steps` dropped.

## Tasks and day plans API (decided 2026-10-23, second pass)

- `GET /tasks?status&important&urgent` → plain `Task[]` (all tasks, no goal, no priority): the clean Tasks endpoint.
- `GET /tasks/goals?status&important&urgent&goal=none` → `TaskWithGoal[]`, `TaskWithGoal = Task & { priority, goal: Goal | null }` (flat, full `Goal`). `goal=none` feeds the «Без цілі» card. Why: mirrors `/goals/tasks`; the server filters instead of the client.
- Filters are the quadrant flags `important` / `urgent`, not a computed quadrant number; all filters optional, omitted = no filter.
- Dropped: `GET /tasks/by-goal`, `GET /tasks?goal=none`, `plannedBefore`.
- `GET /day-plans?startDate&endDate&status&important&urgent` → `DayPlan[] = { id, date, tasks: TaskWithGoal[] }[]`. A day's tasks come only from `task_for_day`. Dates optional (one-sided ranges allowed); neither → server's current UTC date (accepted gap near local midnight; exception to "client decides today"). Days with a plan are returned even if empty (client needs the `id`).
- Past days for the picker («З минулих днів») = `GET /day-plans?endDate=<yesterday>&status=todo`, deduplicated on the client. Why: past-plan data belongs to day plans; rejected a `carryover` endpoint and `plannedBefore` on `/tasks`.
- Grouping the day by goal moved to the client (server returns a flat list); the grouping rules themselves are unchanged.
- `POST /day-plans { date, taskIds? }` creates a plan (may be empty); existing date → `409 CONFLICT`. `PUT /day-plans/:id/add { taskIds }` and `PUT /day-plans/:id/remove { taskIds }` add / remove `task_for_day` rows; all return `DayPlan`. Why `PUT`: both are idempotent. Rejected: upsert on `POST`. Dropped: `/day-plans/:date/items` endpoints.
- Notes re-reviewed: unchanged (no editing, no pagination; `convert` keeps its `{ type, input }` body).
- `TaskWithGoal` everywhere a task comes with its goal, including task mutation responses (`POST /tasks` 201, `PATCH /tasks/:id`, `PATCH /tasks/:id/status`) and `POST /notes/:id/convert` with `type: 'task'`. `TaskItem` removed. Why: one flat shape; a mutation response goes straight into the `/tasks/goals` and day-plan caches.

## Scope

- Postponed: Progress page, Eisenhower matrix page, reflections, AI assistant, release process.

## Deviations from source docs

| Source doc says | Decision |
|---|---|
| `apps/web`, `apps/api`, `packages/shared` | `client/`, `server/`, `shared/`, `http/` |
| Docker Compose with postgres, mongo, api, web | Postgres only in Docker; apps run on the host |
| Integration tests in CI | No tests in the first stages |
| Email/password + Google (`passwordHash`) | Email/password first, Google later |
| «Терміново» set automatically (functional sections) | Never automatic (decisions section wins) |
| Same site via proxy | No proxy; different ports on `localhost` are the same site |
| Day plan shown as quadrant blocks 1–3 (+4 if non-empty), «+ Додати» per quadrant | Day plan grouped by goal («Без цілі» last), «+ Додати» per goal group |

## Amendments to approved sections

- **02 Shared (2026-10-23):** `PAST_DAY_READONLY` removed from `ErrorCode`; `ApiErrorSchema` gets `message` (English, developer-only; UI translates `code`), matching sections 01 and 04.
- **03 Data model (2026-10-31):** `users` has `passwordHash`, no `googleId` for now; index `UNIQUE(email)` replaces `UNIQUE(googleId)`.
- **02 Shared (2026-10-31):** `ErrorCode` gets `INVALID_CREDENTIALS`; `inputs/auth.ts` added.
- **01 Architecture (2026-10-23):** `shared/` split into `shared/` (`@kurs/shared`, domain) and `http/` (`@kurs/http`, contract). Server routes register from `@kurs/http` via `route()`; client `api/` calls `request(route, …)` instead of hand-written descriptors. Details in section 02.
- **01 Architecture (2026-10-31):** client styles are plain CSS + CSS Modules instead of Tailwind v4 (`src/styles/global.css` with CSS variables from the prototype).
- **04 API contract (2026-10-31):** title says stages 1–4 (there are four stages, see section 08).
- **05 Auth (2026-10-31):** `set-password` admin script dropped (removed from sections 05 and 08); no way to recover a forgotten password until reset by email.

## Server internals (decided 2026-10-31, see section 09)

- Settings: one Zod schema in `server/src/env.ts`, parsed at startup, exit on error; only `env.ts` reads `process.env`; `.env` loaded by Node (`--env-file-if-exists`), no `dotenv`. Variables: `NODE_ENV`, `PORT`, `DATABASE_URL`, `CLIENT_URL`, `JWT_SECRET`.
- Errors: services/repositories throw `AppError(code, message, details?)`; one `app.onError` builds `{ code, message, details? }`; Zod failures → `VALIDATION`; unknown route → `NOT_FOUND`; anything else → `500 INTERNAL` with a generic message, real error only in the log. `ErrorCode → status` map lives in the server.
- Logging: Hono `logger()` (one line per request) + `console.error` with stack for 500s; never log bodies, cookies, tokens, passwords. `pino` later, with deployment.
- Transactions: a separate layer of multi-table repositories (`repositories/multi/`) on top of table repositories (`repositories/tables/`). Multi-table repositories may use table repositories and are the only place that starts a transaction; table repositories never use multi-table ones; services may use both and never see `db` / `tx`. Table repositories are factories `createXRepo(db | tx)`. Why: the user wants services free of database code and no "table repository that secretly writes other tables". Rejected: services starting transactions (services would touch `db`), a Unit of Work layer (more machinery than needed), multi-table methods inside table repositories (a notes repository writing `tasks`).
- Refresh rotation without a transaction: conditional `UPDATE … WHERE used = false RETURNING`.
- **01 Architecture (2026-10-31):** `repositories/` split into `tables/` and `multi/`; layer rules updated (section 09).
