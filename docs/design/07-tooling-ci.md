# 07 — Tooling and CI

Status: ✅ approved (2026-10-31)

## Tooling

- Node 24 LTS, pinned in `.nvmrc` and `engines`.
- pnpm workspaces: `server`, `client`, `shared`, `http`.
- TypeScript strict; shared `tsconfig.base.json`; project references.
- `shared` and `http`: built with `tsc -b` into `dist/`, `tsc -b --watch` in dev (section 02).
- Server dev: `tsx watch`; server build: `tsc`. Client dev and build: Vite.
- Biome for lint + format.
- No tests in the first stages.

## Import boundaries (Biome)

Boundaries between packages (`shared`, `http`, `server`, `client`) are enforced by pnpm dependencies and TS project references — see section 02. Biome covers only layers inside `server`, via `noRestrictedImports` with per-folder `overrides`:

- `server/src/routes/**` must not import `repositories` or `db`.
- `server/src/services/**` must not import `routes`, `@kurs/http`, `db` or `drizzle-orm`.
- `server/src/repositories/tables/**` must not import `repositories/multi`.
- `db.transaction` only in `server/src/repositories/multi/**` (section 09; checked by review if Biome cannot express it).

```jsonc
{
  "overrides": [
    {
      "includes": ["server/src/services/**"],
      "linter": { "rules": { "style": { "noRestrictedImports": { "level": "error", "options": {
        "paths": { "@kurs/http": "services work with domain types only" }
      }}}}}
    }
  ]
}
```

Exact rule group and `patterns` syntax to be verified against the current Biome 2.x during setup.

## Local environment

- `docker-compose.yml`: Postgres only.
- `.env.example` committed; `.env` never committed.
- One documented start sequence in `README`: `docker compose up -d` → `pnpm db:migrate` → `pnpm dev`.

## CI (GitHub Actions, on PRs to `development` and `main`)

1. PR title follows Conventional Commits.
2. `biome ci`
3. Typecheck
4. Build

Dependency caching; build only what changed (tooling to decide).

## Git workflow

- `feature/*` from `development` → PR to `development` (squash) → PR `development` → `main` (merge commit).
- No direct pushes to `development` / `main`; merge only with green CI.

## Later

- release-please, changelog, tags, branch protection setup.
- Tests (Vitest, Testcontainers, Playwright).
