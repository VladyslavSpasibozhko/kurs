# Kurs — design

Design for the real implementation of «Курс». Source material lives in `data/`:
functional requirements, infrastructure doc, HTML prototype, architecture and data model diagrams.

## Sections

| # | Section | Status |
|---|---|---|
| 00 | [Decisions log](00-decisions.md) | living document |
| 01 | [Repository and architecture](01-architecture.md) | ✅ approved (amended 2026-10-23) |
| 02 | [Shared packages: `@kurs/shared` + `@kurs/http`](02-shared-and-http.md) | ✅ approved |
| 03 | [Data model](03-data-model.md) | ✅ approved |
| 04 | [API contract](04-api-contract.md) | ✅ approved (second pass 2026-10-23); Progress ⏸ postponed |
| 05 | [Authentication](05-auth.md) | ✅ approved (2026-10-31, email + password; Google later) |
| 06 | [Client](06-client.md) | ✅ approved (2026-10-31, plain CSS + CSS Modules instead of Tailwind) |
| 07 | [Tooling and CI](07-tooling-ci.md) | ✅ approved (2026-10-31) |
| 08 | [Stages](08-stages.md) | ✅ approved (2026-10-31) |
| 09 | [Server internals](09-server-internals.md) | ✅ approved (2026-10-31) |

## Diagrams

Derived from sections 01, 03, 05 and 09 (sources in `.mmd`, re-render with `npx -y @mermaid-js/mermaid-cli -i <file>.mmd -o <file>.png -s 2 -b white`):

- [Backend layers](diagrams/backend-layers.png) — [source](diagrams/backend-layers.mmd)
- [Data model](diagrams/data-model.png) — [source](diagrams/data-model.mmd)

## Not yet discussed

- Per-stage detailed specs and implementation plans.
- Release process (release-please, branch protection) — scheduled for later.
- Postponed features: Google login, email verification, password reset by email, login rate limiting (before deployment), Progress page, Eisenhower matrix page, reflections (MongoDB), AI assistant (MCP), voice input.
