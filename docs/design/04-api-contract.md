# 04 — API contract (stages 1–4)

Status: ✅ approved — Notes, Goals, Tasks and Day plans (2026-10-23, second pass); Progress ⏸ postponed (2026-10-31)

All paths are under `/api`. 🔒 = requires auth (`requireAuth`). Defined in `@kurs/http` (see section 02).

## System

| | |
|---|---|
| `GET /health` | `{ status, db }` — end-to-end check for stage 1 |

## Auth

| | |
|---|---|
| `POST /auth/register` | `RegisterInput { email, password }` → 201 `{ userId }` + session cookies; taken email → `CONFLICT` |
| `POST /auth/login` | `LoginInput { email, password }` → 200 `{ userId }` + session cookies; wrong email or password → `INVALID_CREDENTIALS` |
| `POST /auth/refresh` | rotation → 200 `{ userId }` + new cookies; reuse of a used token → revoke all of the user's tokens → 401 |
| `POST /auth/logout` | revoke refresh token, clear cookies → 204 (always) |
| 🔒 `GET /auth/session` | `{ userId }` or 401 |
| 🔒 `PATCH /auth/password` | `ChangePasswordInput { currentPassword, newPassword }` → 204; wrong current password → `INVALID_CREDENTIALS` |

Details in section 05 (amended 2026-10-31: email + password instead of Google).

## Notes 🔒

| | |
|---|---|
| `GET /notes` | newest first |
| `POST /notes` | `CreateNoteInput { text }` |
| `DELETE /notes/:id` | |
| `POST /notes/:id/convert` | body `{ type: 'task', input: CreateTaskInput } \| { type: 'goal', input: CreateGoalInput }` → creates the task or goal, deletes the note (one transaction); returns `{ type: 'task', task: TaskWithGoal } \| { type: 'goal', goal: GoalWithTasks }` (`goal` has the same shape as one item of `GET /goals/tasks`). With `type: 'task'` and `input.planDate` the task goes straight into that day's plan (used by the picker's «Нотатки» tab) |

`convert` body and response are Zod discriminated unions on `type`. The inputs are nested under `input` and reused unchanged from `@kurs/shared/inputs` (not flattened next to `type`). The server dispatches on `type` to `NoteService.convertToTask` / `convertToGoal`; the client knows `type` when calling, so it invalidates notes plus tasks/day plan or goals.

## Task shapes

Defined in `@kurs/shared/domain`:

```ts
Task          // plain task: no goalId, no priority
TaskWithGoal  = Task & { priority: StepPriority | null, goal: Goal | null }   // a task with its goal: task lists, day plan, task mutations
Goal          // plain goal; no `progress` for now (postponed with the Progress section)
StepItem      = Task & { priority: StepPriority | null }   // a task inside its goal: flat, no `goal` (the parent is the goal)
GoalWithTasks = Goal & { tasks: StepItem[] }
DayPlan       = { id, date: IsoDate, tasks: TaskWithGoal[] }
```

All shapes are flat (`...task` + extra fields). A task has three shapes: `Task` (plain), `StepItem` (inside a goal), `TaskWithGoal` (with its goal: lists, day plan, task mutations). `TaskItem` is removed (2026-10-23).

## Goals 🔒

The Goals page makes two parallel requests: `GET /goals/tasks?status=active&taskStatus=todo` (goal cards) and `GET /tasks/goals?goal=none&status=todo` («Без цілі» card). Each has its own cache key; a mutation invalidates only the lists it touches.

| | |
|---|---|
| `GET /goals?status=active\|done` | `Goal[]` (no tasks) sorted by `createdAt`; `status` optional, omitted = all goals. For the goal dropdown in the task form |
| `GET /goals/tasks?status=active\|done&taskStatus=todo\|done` | `GoalWithTasks[]` sorted by goal `createdAt`; both params optional, omitted = no filter (all goals / all steps); `taskStatus` filters the steps shown. Registered before `/goals/:id` routes |
| `POST /goals` | `CreateGoalInput { title, description?, dueDate?, steps?: CreateStepInput[] }` → `GoalWithTasks` (201) |
| `PATCH /goals/:id` | `UpdateGoalInput { title?, description?, dueDate? }` (no `status`) → `GoalWithTasks` (200) |
| `PATCH /goals/:id/status` | `{ status: 'done' } \| { status: 'active' }` → `done` finishes the goal manually and sets `completedAt` (server time); `active` reopens it and clears `completedAt`. Returns `GoalWithTasks` (200). Same convention as `PATCH /tasks/:id/status` |
| `DELETE /goals/:id?steps=keep\|delete` | `steps` is required: `keep` → steps become tasks without a goal; `delete` → steps are deleted too (see section 03) |

## Tasks 🔒

Filters on both lists are optional; omitted = no filter. `important` / `urgent` are `true|false` (the quadrant flags). Both lists sorted by `createdAt`. Registered before `/tasks/:id` routes.

| | |
|---|---|
| `GET /tasks?status=todo\|done&important&urgent` | `Task[]` — every task, plain: no `goal`, no `priority` |
| `GET /tasks/goals?status=todo\|done&important&urgent&goal=none` | `TaskWithGoal[]` — every task with its step `priority` and full `goal` (`null` = no goal). `goal=none` → only tasks without a goal («Без цілі» card) |
| `POST /tasks` | `CreateTaskInput { title, dueDate?, durationMin?, description?, important, urgent, goalId?, priority?, planDate? }` → `TaskWithGoal` (201); `planDate` adds it straight to that day's plan (creating the plan if needed); with `goalId` it is created as a step of that goal (used by «+ Додати» in the day plan) |
| `PATCH /tasks/:id` | `UpdateTaskInput` → `TaskWithGoal` (200); changing goal moves/removes the `task_in_goal` row |
| `PATCH /tasks/:id/status` | `{ status: 'done', date } \| { status: 'todo' }` → `done` sets `doneAt = date`, `todo` clears `doneAt`; returns `TaskWithGoal` (200). Replaces `complete` / `reopen` |
| `DELETE /tasks/:id` | |

Dropped (2026-10-23): `GET /tasks/by-goal` (grouped), `GET /tasks?goal=none`, `GET /tasks?plannedBefore=` (past days now come from `GET /day-plans`).

## Day plans 🔒

A day's tasks are exactly its `task_for_day` rows: `day_plans` (`userId`, `day`) → `task_for_day` → `tasks`. `dueDate` / `doneAt` never decide which tasks a day contains. One plan per day (`UNIQUE(userId, day)`).

| | |
|---|---|
| `GET /day-plans?startDate&endDate&status=todo\|done&important&urgent` | `DayPlan[]` sorted by `date` ascending; tasks inside a day by `createdAt`. All params optional: only `startDate` → that day onward; only `endDate` → up to that day; both inclusive; neither → the server's current UTC date (see "What is today"). `status` / `important` / `urgent` filter the tasks (current task status). Every day that has a plan is returned, even with `tasks: []` |
| `POST /day-plans` | `{ date, taskIds? }` → creates the plan (may be empty) and a `task_for_day` row per task, one transaction → `DayPlan` (201). A plan for that date already exists → `409 CONFLICT` (client uses `/add`) |
| `PUT /day-plans/:id/add` | `{ taskIds }` → creates `task_for_day` rows; tasks already in the plan are skipped → `DayPlan` (200) |
| `PUT /day-plans/:id/remove` | `{ taskIds }` → deletes those `task_for_day` rows; the tasks stay → `DayPlan` (200) |

- `PUT` because both are idempotent (repeating gives the same result).
- Unknown or another user's `taskIds` → `NOT_FOUND`, nothing changed (all or nothing).
- "Done on that day" = `rules.isDoneOn(task, date)` (`task.doneAt === date`), decided on the client; the client also decides whether a day is past (read-only).

Dropped (2026-10-23): `GET /day-plans/:date` (grouped), `POST /day-plans/:date/items`, `DELETE /day-plans/:date/items/:taskId`, the `carryover` idea.

**Grouping is a client rule** (the server returns a flat `tasks` list):
- Groups by goal, ordered by goal `createdAt`; «Без цілі» (`goal: null`) last; only non-empty groups.
- Tasks inside a group: by step `priority` (high → low, null last), then `createdAt`.
- Quadrant stays a task attribute (`important` / `urgent`); shown per task via `rules.quadrantOf`, the plan is not split into quadrant blocks.

**Day page flow:** `GET /day-plans?startDate=D&endDate=D` → no plan: picker saves with `POST /day-plans { date: D, taskIds }`; plan exists: `PUT /day-plans/:id/add { taskIds }`.

**«Спланувати день» picker** reads from existing lists, one tab per source:

| Tab | Request |
|---|---|
| «З минулих днів» | `GET /day-plans?endDate=<D − 1>&status=todo`; the client flattens and removes duplicates by task id |
| «Цілі» | `GET /goals/tasks?status=active&taskStatus=todo` (steps sorted by priority) |
| «Без цілі» | `GET /tasks/goals?goal=none&status=todo` |
| «Нотатки» | `GET /notes` |

- Tabs share the cache with the Goals and Notes pages.
- The client hides tasks already in the day's plan; a task may appear in two tabs (selection is by task id).
- Selected tasks → one `POST /day-plans` or `PUT /day-plans/:id/add`.
- A note is added on its own button with a chosen quadrant → `POST /notes/:id/convert` with `{ type: 'task', input: { …, planDate } }` (task created, note deleted, right away).
- «+ Додати» in a goal group → `POST /tasks` with `planDate` and that group's `goalId` (plus `priority`); in the «Без цілі» group → `POST /tasks` with `planDate` only. Quadrant is set in the form via `important` / `urgent`.

## Progress ⏸ postponed

Postponed 2026-10-31: implementation starts without the Progress page; it is a future stage. No progress routes in `@kurs/http` for now.

Idea for when it is picked up (not approved): no `/progress/*` endpoints, reuse existing lists —
done this month via `GET /tasks/goals?status=done&doneFrom&doneTo` (new filters), goals via `GET /goals/tasks?status=active`
+ `rules.goalProgress`, open tasks by quadrant via `GET /tasks?status=todo` + `rules.quadrantOf`.

## Errors

Shape: `{ code, message, details? }` (`ApiErrorSchema` in `@kurs/http`). `message` is short English text for developers (devtools, logs), e.g. `"Task 3f2a… not found"`; the UI never shows it and translates `code` via i18n.

| Code | HTTP | When |
|---|---|---|
| `VALIDATION` | 400 | Zod validation failed; `details` = field paths |
| `UNAUTHORIZED` | 401 | no/expired access token |
| `INVALID_CREDENTIALS` | 401 | login or change password: wrong email or password |
| `NOT_FOUND` | 404 | missing, or belongs to another user (existence is not revealed) |
| `CONFLICT` | 409 | e.g. duplicate |
| `INTERNAL` | 500 | anything else |

## What is "today"

The client decides (decided 2026-10-23). It sends dates as its local `YYYY-MM-DD`; the server stores them as given and does not know the user's time zone.

- Past days being read-only is a UI rule only; the server does not reject changes to past days.
- `PATCH /tasks/:id/status { status: 'done', date }` uses the client's date as is.
- Exception (decided 2026-10-23): `GET /day-plans` with no dates defaults to the server's current **UTC** date. Accepted gap: near local midnight it can be yesterday. The client always sends dates; the default is only for manual calls and later MCP.
