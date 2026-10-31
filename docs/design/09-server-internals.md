# 09 — Server internals

Status: ✅ approved (2026-10-31)

How the server reads its settings, turns errors into responses, logs, and runs transactions. Needed from stage 1.

## Settings (`server/src/env.ts`) ✅

- One Zod schema, parsed once at startup. Missing or invalid variable → print which one and why, exit with code 1.
- The rest of the server imports `env` from `env.ts`; nothing else reads `process.env`.
- `.env` is loaded by Node itself (`--env-file-if-exists=.env`), no `dotenv` dependency. Exact flag passing through `tsx watch` verified during setup.

| Variable | Rule | Example |
|---|---|---|
| `NODE_ENV` | `development` \| `production`, default `development` | |
| `PORT` | number, default `3000` | |
| `DATABASE_URL` | URL | `postgres://kurs:kurs@localhost:5432/kurs` |
| `CLIENT_URL` | URL (CORS origin) | `http://localhost:5173` |
| `JWT_SECRET` | at least 32 characters (section 05) | |

Client: `VITE_API_URL` only (`http://localhost:3000/api`).

Not in `.env`: token lifetimes, scrypt settings — code constants (section 05).

## Errors ✅

- `server/src/errors.ts`: `class AppError extends Error { code: ErrorCode; details? }`. Services and repositories throw it: `throw new AppError('NOT_FOUND', \`Task ${id} not found\`)`.
- One `app.onError` handler turns every error into `{ code, message, details? }`:

| Error | Response |
|---|---|
| `AppError` | status from the code (table in section 04), its `code` / `message` / `details` |
| Zod validation failed (`zValidator` hook) | `400 VALIDATION`, `details` = field paths and issue codes |
| Unknown route (`app.notFound`) | `404 NOT_FOUND` |
| Anything else | `500 INTERNAL`, message `"Internal error"`; the real error is logged, never sent |

- `ErrorCode → HTTP status` map lives in the server (HTTP is not a domain concern).
- Known Postgres errors are translated in repositories, e.g. unique violation `23505` on register → `AppError('CONFLICT')` (section 05).

## Logging ✅

- Hono's built-in `logger()` middleware: one line per request (method, path, status, time).
- `500` errors: `console.error` with the stack in `onError`.
- Never logged: request bodies, cookies, tokens, passwords.
- No logging library for now. Possible later (deployment): `pino` with JSON output.

## Transactions ✅

Multi-table work gets its own layer: **multi-table repositories** on top of **table repositories**.

```
routes → services → multi-table repositories → table repositories → db
                  ↘ table repositories ↗
```

| Layer | Folder | Does | May use |
|---|---|---|---|
| Table repositories | `repositories/tables/` | one table each: `notes`, `tasks`, `goals`, `day-plans`, `users`, `refresh-tokens` | `db` |
| Multi-table repositories | `repositories/multi/` | one action over several tables, each method in one transaction | table repositories, `db.transaction` |
| Services | `services/` | decisions and checks | both kinds of repositories |

Rules (enforced by Biome):
- Table repositories never import multi-table repositories.
- Only multi-table repositories start transactions (`db.transaction`).
- Services never import `db/`, `drizzle-orm` or `tx`.
- Repositories do data steps only; decisions stay in services (e.g. which goal-delete mode).

Table repositories are created from a connection, so a multi-table repository can run them inside its transaction:

```ts
// repositories/tables/tasks.repo.ts
export const createTasksRepo = (db: DbOrTx) => ({
  create(userId, input) { … },
});
export const tasksRepo = createTasksRepo(db);   // services use this (bound to db)

// repositories/multi/note-conversion.repo.ts
toTask(userId, noteId, input) {
  return db.transaction(async (tx) => {
    const task = await createTasksRepo(tx).create(userId, input);
    await createNotesRepo(tx).delete(userId, noteId);
    return task;
  });
}
```

Multi-table actions in Курс: note → task / goal, create goal with steps, add or move a task to a goal, delete goal (`keep` / `delete`), create a day plan with tasks, create a task straight into a day. Examples of multi-table repositories: `note-conversion`, `goal-steps`, `day-planning`; the exact list is decided in the stage 3 plan.

Refresh token rotation needs no transaction: "mark used only if not used" is one `UPDATE … WHERE used = false RETURNING`, then the new token is inserted.

## Not now

- Graceful shutdown (closing the DB pool on `SIGTERM`) — with deployment.
- Request IDs, structured logs — with `pino`.
