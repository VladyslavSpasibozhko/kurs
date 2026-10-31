---
name: kurs-design-in-progress
description: "Design for «Курс» is approved (sections 01–09 in docs/design); next is the detailed spec and plan for stage 1"
metadata:
  node_type: memory
  type: project
  originSessionId: 1de64cab-ba11-427e-b141-37db17f99d3e
  modified: 2026-10-31T16:00:00.000Z
---

Design for «Курс» (personal goals/tasks app; source docs referred to as `data/`, not in this repo). No code written yet.

- All sections 01–09 in `docs/design/` are approved (last on 2026-10-31). `README.md` lists them; `00-decisions.md` holds every decision with its reason and the amendments to approved sections.
- Current shape in brief: pnpm workspace with `server` (Node 24 + Hono, Drizzle/Postgres), `client` (React + Vite, TanStack Router/Query, plain CSS + CSS Modules), `@kurs/shared` (domain) and `@kurs/http` (route contract). Email + password auth (scrypt, access JWT + rotating refresh token); Google later. Task shapes: `Task`, `StepItem`, `TaskWithGoal`, `GoalWithTasks`, `DayPlan`. Day plan grouped by goal on the client. Transactions only in `repositories/multi/`.
- Postponed: Progress page, Eisenhower matrix, reflections (MongoDB), AI assistant (MCP), Google login, email verification, password reset, rate limiting (before deployment), release process, tests.

**Why:** the user designs section by section and approves each before any code.
**How to apply:** read `docs/design/README.md`; next step is the detailed spec + implementation plan for stage 1 (Foundation, section 08). Don't start implementation until that plan is approved.
