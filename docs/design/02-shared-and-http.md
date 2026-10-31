# 02 — Shared packages: `@kurs/shared` and `@kurs/http`

Status: ✅ approved

## Two packages

| Package | Folder | Answers | Depends on |
|---|---|---|---|
| `@kurs/shared` | `shared/` | What the data is (domain) | `zod` |
| `@kurs/http` | `http/` | How it travels over HTTP (contract) | `@kurs/shared`, `zod` |

```
shared  ←  http  ←  client
   ↑         ↑
   └──── server     (services use only shared; routes use http)
         MCP tools later: only shared
```

The domain knows nothing about HTTP: no paths, no query-string coercion, no status codes.

## `@kurs/shared` — domain

```
shared/src/
  domain/
    common.ts       Id, IsoDate ('2026-10-22'), IsoDateTime, DurationMinutes
    note.ts         NoteSchema
    task.ts         TaskSchema (plain), TaskWithGoal { ...task, priority | null, goal: Goal | null }, StepPriority, Quadrant
    goal.ts         GoalSchema (no progress for now), StepItem (Task + priority), GoalWithTasks { ...goal, tasks: StepItem[] }
    day-plan.ts     DayPlanSchema { id, date, tasks: TaskWithGoal[] }
  inputs/
    note.ts, task.ts, goal.ts, day-plan.ts, auth.ts (EmailSchema, PasswordSchema, RegisterInput, LoginInput, ChangePasswordInput)
                    CreateXInput, UpdateXInput — what services and MCP tools accept
  rules/
    quadrant.ts     quadrantOf(task)
    progress.ts     goalProgress(steps)        — pure functions, no I/O
  errors.ts         ErrorCode: VALIDATION, UNAUTHORIZED, INVALID_CREDENTIALS, NOT_FOUND, CONFLICT, INTERNAL
```

Exports (subpaths, no barrel): `@kurs/shared/domain`, `@kurs/shared/inputs`, `@kurs/shared/rules`, `@kurs/shared/errors`.

## `@kurs/http` — contract

```
http/src/
  define-route.ts   defineRoute(), RouteDef, SuccessBody<R>
  api-error.ts      ApiErrorSchema { code: ErrorCode, message, details? }  (message: English, for developers only; the UI translates code)
  routes/
    system.ts, auth.ts, notes.ts, tasks.ts, goals.ts, day-plans.ts
```

A route is method + path + path params + query (coerced from strings) + body + one success response. Body and response reuse schemas from `@kurs/shared`.

### Route definition

```ts
type RouteDef = {
  method: 'GET' | 'POST' | 'PATCH' | 'PUT' | 'DELETE';
  path: string;                 // '/tasks/:id'
  params?: z.ZodObject;
  query?: z.ZodObject;
  body?: z.ZodType;
  response?: z.ZodType;         // absent → no body
  status?: 200 | 201 | 204;     // default: GET/PATCH/PUT 200, POST 201, DELETE 204
};

// Returns the definition unchanged, keeping literal types
export const defineRoute = <const R extends RouteDef>(r: R) => r;

export type SuccessBody<R extends RouteDef> =
  R['response'] extends z.ZodType ? z.infer<R['response']> : undefined;
```

- One success response per route. Errors are not listed per route: every error is `ApiErrorSchema` with a `code`.
- `params` are written explicitly (not inferred from the path string).

### Example

```ts
// http/src/routes/tasks.ts
import { Id, TaskWithGoal } from '@kurs/shared/domain';
import { UpdateTaskInput } from '@kurs/shared/inputs';

export const updateTask = defineRoute({
  method: 'PATCH', path: '/tasks/:id',
  params: z.object({ id: Id }),
  body: UpdateTaskInput,
  response: TaskWithGoal,
});
```

## How the contract is used

The server registers its routes **from** the contract, so it cannot drift from it.

```ts
// server: method, path and validators come from the contract;
// the handler must return SuccessBody<typeof updateTask>
route(app, updateTask, async ({ params, body, userId }) =>
  taskService.update(userId, params.id, body));

// client: path, method and return type come from the contract
request(updateTask, { params: { id }, body });   // → Promise<TaskWithGoal>
```

| Helper | Lives in | Does |
|---|---|---|
| `route(app, def, handler)` | `server/src/routes/route.ts` | registers `def.method` + `def.path`, attaches `zValidator` for params/query/body, puts `userId` from `requireAuth` into the handler args, types the handler's return as `SuccessBody<def>`, replies with the default status |
| `request(def, { params, query, body })` | `client/src/api/http.ts` | substitutes `:params`, appends the query string, sends JSON with `credentials: 'include'`, refreshes once on 401, throws `ApiError` on error, returns `SuccessBody<def>` |

The client does not validate responses at runtime; the types come from the contract.

What is checked at compile time:

| Level | Checked by |
|---|---|
| Request data (body, query) | Schemas from `@kurs/shared` on both sides |
| Response data | `SuccessBody<def>` on both sides |
| Which path and method go with which schemas | `@kurs/http` — the server registers from it, the client calls through it |

## Schema rules

- Inputs are derived from entities (`pick` / `omit` / `partial`), not written twice.
- No `userId` anywhere in `shared` or `http` — the server takes it from the token.
- Dates are strings (`IsoDate`, `IsoDateTime`), never `z.date()`.
- Cross-field rules live in the schema (`refine` / `superRefine`).
- No user-facing messages in schemas; errors carry codes and paths, the client translates.
- Naming: `XxxSchema` (schema), `Xxx` (type), `XxxInput` (input). `z.infer` for output types, `z.input` for what the client sends.
- Query-string coercion (`z.coerce`) lives only in `@kurs/http`.
- No `{ data: … }` envelope in responses.
- DB schemas are in neither package; they live in `server/src/db`.

## Boundaries

| Boundary | Enforced by |
|---|---|
| `shared` must not import `http`, `server`, `client` | pnpm dependencies + TS project references |
| `http` must not import `server`, `client` | pnpm dependencies + TS project references |
| `client` must not import `server` | pnpm dependencies + TS project references |
| server: `routes` → `services` → `repositories` | Biome `noRestrictedImports` |
| server `services` must not import `@kurs/http` | Biome `noRestrictedImports` |

pnpm only links a package's declared dependencies into its `node_modules`, so an undeclared `@kurs/*` import fails with TS2307. Requirements: the root `package.json` holds only dev tooling (never `@kurs/*`), and `.npmrc` has no `shamefully-hoist` / `node-linker=hoisted`. Relative imports into another package fail `tsc -b` with TS6307 because each package's `tsconfig` covers only its own `src/`.

## Build

Each package builds itself and exposes compiled code. The server and client use `@kurs/shared` and `@kurs/http` like any regular package.

- `shared` and `http` compile with `tsc -b` into `dist/`: `.js`, `.d.ts` and `.d.ts.map` (declaration maps, so go-to-definition in the editor opens the source).
- `exports` point at `dist/`:
  ```jsonc
  // shared/package.json
  "type": "module",
  "exports": {
    "./domain": { "types": "./dist/domain/index.d.ts", "default": "./dist/domain/index.js" },
    "./inputs": { "types": "./dist/inputs/index.d.ts", "default": "./dist/inputs/index.js" },
    "./rules":  { "types": "./dist/rules/index.d.ts",  "default": "./dist/rules/index.js" },
    "./errors": { "types": "./dist/errors.d.ts",       "default": "./dist/errors.js" }
  }
  ```
- ES modules, `module: nodenext`: the output runs on plain Node, so relative imports inside the packages are written with the `.js` extension (`import { Id } from './common.js'`).
- `dist/` is git-ignored.
- Build order comes from project references and pnpm: `shared` → `http` → `server` / `client`.

| Command | Does |
|---|---|
| `pnpm build` | builds everything in dependency order (`shared`, `http`, then `server` with `tsc`, `client` with Vite) |
| `pnpm dev` | builds `shared` and `http` once, then runs in parallel: `tsc -b --watch` for both packages, `tsx watch` for the server, Vite for the client |

- After a change in `shared` or `http`, the watcher rebuilds `dist/`; the server restarts and the client reloads from the new `dist/`. Without `pnpm dev` running, the package must be rebuilt (`pnpm build`) before its change is visible.
- The server's production build is plain `tsc` into `server/dist/`; no bundler is needed because the packages are already JavaScript.
- To verify during setup: `tsx watch` and Vite pick up changes in a linked package's `dist/`.
