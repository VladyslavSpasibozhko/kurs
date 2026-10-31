# 03 — Data model (Postgres)

Status: ✅ approved (2026-10-23) · amended 2026-10-31 (`users`: `passwordHash`, no `googleId`)

Based on `data/data_model_tables_with_names.png`, with fields the requirements need added (➕).
Defined in `server/src/db/schema.ts` (Drizzle). IDs are UUIDs. Column names in camelCase, as in the diagram.

## Tables

**users**
- `id`, `email` (unique, stored normalized: trimmed + lowercased), `passwordHash` (`scrypt$N=…,r=…,p=…$<salt>$<hash>`), `createdAt`
- No `googleId` for now (amended 2026-10-31: email + password first). Google later: migration adds `googleId text UNIQUE` (nullable) and makes `passwordHash` nullable.

**refresh_tokens**
- `tokenHash` (PK), `userId`, `used`, `expiresAt`, ➕`createdAt`

**notes**
- `id`, `userId`, `text`, `createdAt`

**goals**
- `id`, `userId`, `title`, `status` (`active` | `done`), `createdAt`
- ➕`description` (nullable)
- ➕`dueDate` (nullable)
- ➕`completedAt` (nullable) — set when the user finishes the goal manually, cleared when it is reopened

**tasks**
- `id`, `userId`, `title`, `status` (`todo` | `done`), `important`, `urgent`, `dueDate` (nullable), `createdAt`, `updatedAt`
- ➕`description` (nullable)
- ➕`durationMin` (nullable) — for "time remaining" in the day plan
- ➕`doneAt` (nullable) — for "done this month"

**task_in_goal**
- `id`, `goalId`, `taskId` (**UNIQUE** — a task is in at most one goal), `priority` (`high` | `medium` | `low`, nullable; renamed from `weight`)
- No row → task has no goal and no step priority. There is no system "Without goal" goal.

## Tables vs domain

The API returns domain objects from `@kurs/shared`, not table rows. The domain `Task` is plain (no `goalId`, no `priority`)
and is what `GET /tasks` returns. Everywhere a task comes with its goal (task-with-goal lists, day plan, task mutations)
the tasks repository reads it with `LEFT JOIN task_in_goal` and returns `TaskWithGoal { ...task, priority, goal: Goal | null }`
(`goal` null → no goal, `priority` null without a goal).
`TaskService` decides what `goalId` / `priority` from `CreateTaskInput` / `UpdateTaskInput` mean (insert, move or delete
the `task_in_goal` row); a multi-table repository writes the task and that row in one transaction (section 09).

**day_plans**
- `id`, `userId`, `day` (date), ➕**UNIQUE(`userId`, `day`)**

**task_for_day**
- `id`, `dayId`, `taskId`, ➕**UNIQUE(`dayId`, `taskId`)**
- No `finished` column (removed 2026-10-23): "done on day D" is derived as `tasks.doneAt = D`.

## Rules

- Every repository query filters by `userId`.
- Quadrant is never stored; computed from `important` / `urgent` (`@kurs/shared/rules`).
- Goal progress is never stored; computed from its steps.
- «Терміново» is never set automatically.
- A goal changes status only by user action.

## Dates and times

| Kind | Columns | Postgres | JSON / `@kurs/shared` |
|---|---|---|---|
| Day (which day it is for) | `day_plans.day`, `tasks.dueDate`, `tasks.doneAt`, `goals.dueDate` | `date` | `IsoDate` string `'2026-10-24'` |
| Moment (when it happened) | `createdAt`, `updatedAt`, `goals.completedAt`, `refresh_tokens.expiresAt` | `timestamptz` (UTC) | `IsoDateTime` string `'2026-10-23T08:00:00Z'` |

- Days are never converted between time zones; the client sends its local date as is.
- Not numbers: a day stored as a timestamp changes when read in another time zone or across DST.

## Done on a day

Only `tasks.status` and `tasks.doneAt` record completion; `task_for_day` has no `finished` flag.
- Mark done on day D → `status = done`, `doneAt = D`. Reopen → `status = todo`, `doneAt = null`.
- A plan item counts as done that day when `task.doneAt === day` (`@kurs/shared/rules` `isDoneOn(task, day)`).
- A task planned on several days shows done only on the day it was finished; earlier days show it not done.
- Trade-off: history follows the task's current state. Done on the 20th, reopened, done again on the 24th → the 20th now shows not done. Accepted: reopening is rare, and nothing needs syncing.

## Deleting a goal

The user chooses one of two modes (`GoalService.delete(userId, goalId, mode)`; the data steps run in one transaction in a multi-table repository, section 09):

| Mode | Removes | Keeps |
|---|---|---|
| `keep` | the goal, its `task_in_goal` rows | the tasks; they become tasks without a goal (step priority is gone with the row) |
| `delete` | the goal, its `task_in_goal` rows, its tasks (and their `task_for_day` rows) | — |

- With `delete`, the deleted tasks also disappear from past days' history and from "done this month". The UI says so in the confirmation dialog.
- Foreign keys: `task_in_goal.goalId` and `task_in_goal.taskId` → `ON DELETE CASCADE`; `task_for_day.taskId` → `ON DELETE CASCADE`. In `delete` mode the tasks are deleted explicitly (multi-table repository); cascades only clean up join rows.

## Indexes

Postgres does not index foreign-key columns automatically, so every index is listed explicitly.

| Table | Index | Used for |
|---|---|---|
| `users` | `UNIQUE(email)` | finding the user at login; one account per email |
| `refresh_tokens` | PK `tokenHash`; `(userId)` | refresh lookup and logout; revoking all of a user's tokens (reuse detection) |
| `notes` | `(userId, createdAt DESC)` | notes list, newest first |
| `goals` | `(userId, status)` | active goals (Goals page, goal dropdown) |
| `tasks` | `(userId, status)` | task lists filtered by status, «Без цілі» card |
| `tasks` | `(userId, doneAt)` | "done this month" (Progress, postponed) |
| `task_in_goal` | `UNIQUE(taskId)`; `(goalId)` | a task's goal, "no goal" check; a goal's steps, goal deletion |
| `day_plans` | `UNIQUE(userId, day)` | one plan per day; lookup by date |
| `task_for_day` | `UNIQUE(dayId, taskId)`; `(taskId)` | a day's items; cascade on task delete, open tasks planned on past days |
