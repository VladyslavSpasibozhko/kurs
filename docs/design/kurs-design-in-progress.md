---
name: kurs-design-in-progress
description: "Planning of the «Курс» app is paused mid-design; design sections live in docs/design, sections 01–03 approved"
metadata:
  node_type: memory
  type: project
  originSessionId: 1de64cab-ba11-427e-b141-37db17f99d3e
  modified: 2026-10-22T13:51:13.937Z
---

Design for «Курс» (personal goals/tasks app; source docs in `data/`) was paused on 2026-10-22, before any code was written. Planning only; user wants to resume the section-by-section review later.

- `docs/design/00-decisions.md` holds every decision made so far and its reason (Hono, shared Zod package, Drizzle, Google-only auth via openid-client, TanStack Router/Query, plain CSS + CSS Modules, Biome, no tests yet).
- Approved: section 01 (architecture, amended 2026-10-23) and section 02 (`@kurs/shared` = domain, `@kurs/http` = contract; `defineRoute` with one success response; server registers routes from the contract via `route()`, client calls `request(route, …)`). Build: shared/http compile with `tsc -b` to `dist/`, consumed as regular packages. Section 03 (data model) approved 2026-10-23. Drafts to review: 05–08.
- Decided 2026-10-23: dates are strings (days `YYYY-MM-DD`, moments ISO/UTC); the client decides "today", server does not enforce past-day read-only. Section 04 approved except Progress: day-plan picker reuses existing lists (no candidates endpoint), `POST /day-plans/:date/items { taskIds }` only, notes convert via one `POST /notes/:id/convert` (`{ type, input }` union; replaces `to-task`/`to-goal`), to a day with `input.planDate`; errors `{ code, message, details? }`. Tasks: `task_for_day.finished` removed (done on D = `doneAt === D`); `PATCH /tasks/:id/status` replaces complete/reopen; plain `Task` + `TaskItem { task, goal | null, priority }` everywhere; new `GET /tasks/by-goal`; `GET /goals` keeps its tasks. Day plan grouped by goal (not quadrant): `{ date, groups: [{ goal | null, tasks }] }`.

**Why:** the user explicitly said to pause and discuss the remaining sections later.
**How to apply:** when resuming, read docs/design/README.md and continue with section 05 (auth); don't start implementation until the spec is approved and a plan is written.

**Update 2026-10-23 (API second pass):** Goals, Tasks and Day plans re-designed and approved — see the "Goals API" and "Tasks and day plans API" sections in `00-decisions.md`. Shapes are flat: `Task`, `StepItem`, `TaskWithGoal`, `GoalWithTasks`, `DayPlan`; `TaskItem` removed. Notes re-reviewed, unchanged. Section 04 approved again except Progress.

**Update 2026-10-31:** Progress page postponed to a future stage (no `/progress/*` routes, no `ProgressSummary`); section 04 is fully approved. Next: review 05 Auth, then 06–08.

**Update 2026-10-31 (auth):** Section 05 approved — email + password first (Google later). scrypt via `node:crypto`, open sign-up, register logs in, access JWT 15 min + refresh 30 days sliding with rotation/reuse detection, client-only parallel-refresh fix, refresh tokens never deleted, `PATCH /auth/password` without revoking sessions. Postponed to "Later": Google login, email verification, password reset by email, rate limiting (before deployment). Sections 01–04 amended accordingly. Next: review 06 Client, then 07 Tooling/CI, 08 Stages.

**Update 2026-10-31 (client):** Section 06 approved. Tailwind dropped → plain CSS + CSS Modules (prototype `data/Курс.html` is plain CSS; colours/sizes as CSS variables in `styles/global.css`). Data refresh: invalidate whole resources, no manual cache writes; exact keys, optimistic checkbox and style porting decided per stage. Next: 07 Tooling/CI, 08 Stages (fix "three packages" → four), then a new section on server internals (env, error handler, logging, transactions).

**Update 2026-10-31 (tooling):** Section 07 approved as drafted. Client folder structure in 06 marked preliminary — discussed when frontend work starts. Next: 08 Stages, then the server internals section.

**Update 2026-10-31 (stages):** Section 08 approved; stage 1 says four packages; 04 title fixed to stages 1–4. `set-password` admin script dropped (05, 08, decisions log); no password recovery until reset by email. All sections 01–08 approved. Next: new section 09 on server internals (env validation, error handler, logging, transactions), then per-stage specs/plans starting with stage 1.

**Update 2026-10-31 (server internals):** Section 09 written and approved: env via Zod, `AppError` + one `onError`, Hono `logger()`, transactions in a separate layer of multi-table repositories (`repositories/multi/`, uses `repositories/tables/`; services use both). Section 01 amended. All sections 01–09 approved. Next: detailed spec + implementation plan for stage 1 (Foundation).
