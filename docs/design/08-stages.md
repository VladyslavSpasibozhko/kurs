# 08 — Stages

Status: ✅ approved (2026-10-31)

Each stage gets its own detailed spec, plan and implementation, and ends with something working.

1. **Foundation** — four packages (`server`, `client`, `shared`, `http`), Docker Compose with Postgres, Biome, CI, `/health` end to end
   (contract in `@kurs/http` → Hono route via `route()` → client calls it via `request()` and shows the status).
2. **Auth (email + password)** — register, login, change password, cookie session with refresh rotation, `/auth/session`, protected routes, data isolation by `userId`.
3. **Notes, goals, tasks** — CRUD, «Цілі» page with «Без цілі» card, steps with priority, note → task / goal.
4. **Day plan** — «День» page, picker with four sources, goal groups («Без цілі» last), past-day history.

## Later

- Google login
- Email verification (link by email, `users.emailVerifiedAt`; needs an email service)
- Password reset by email link (one-time token, hash stored, ~30 min; same email service as verification)
- Rate limiting on login/register — **required before public deployment** (in-memory, per IP + email, `429`)
- Progress page (monthly summary, goal progress, open tasks by quadrant, modals)
- Eisenhower matrix page (drag and drop)
- Reflections (+ MongoDB)
- AI assistant (MCP server + tools, confirm/cancel)
- Release process (release-please)
- Tests
- Voice input, AI analysis of reflections, "done per day" chart
