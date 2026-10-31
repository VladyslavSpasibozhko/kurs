# 06 — Client

Status: ✅ approved (2026-10-31)

## Stack

- React + Vite + TypeScript
- TanStack Router (file-based, `@tanstack/router-plugin` for Vite)
- TanStack Query on top of our own `request()`
- Plain CSS + CSS Modules (built into Vite): `styles/global.css` holds the prototype's colours, sizes and fonts as CSS variables plus base styles; each component has its own `*.module.css` next to it
- react-i18next, English only for now (`en` is default and fallback)
- Native elements instead of a component library: `<dialog>` for modals, `<input type="date">`, `<select>`, `<input type="checkbox">`

## API layer

```
http/src/routes/notes.ts     listNotes = defineRoute({ method: 'GET', path: '/notes', query, response })  ← contract (@kurs/http)
client/src/api/http.ts       request(route, { params, query, body }) → fetch → SuccessBody<route>          ← only place with fetch
client/src/api/notes.queries useQuery({ queryKey: ['notes'], queryFn: () => request(listNotes) })
components                   const { data: notes } = useNotes();
```

`request()` responsibilities:
1. Build the URL: substitute `:params`, append query string.
2. `fetch(VITE_API_URL + path, { credentials: 'include' })`; JSON body with `Content-Type`.
3. On 401: call `POST /auth/refresh` once and retry. One shared in-flight refresh promise: parallel 401s wait for the same refresh.
4. Errors → `ApiError { status, code, details }`; UI translates `code` via i18n.
5. Success → parsed JSON typed as `SuccessBody<route>`; no response schema → `undefined`. No runtime validation of responses.

## Routes

| Path | Screen |
|---|---|
| `/login` | email + password |
| `/register` | email + password; logs in on success |
| `_auth/account` | change password |
| `_auth` layout | checks `/auth/session` before rendering children |
| `/notes` | Notes |
| `/goals` | Goals (with «Без цілі» card) |
| `/day/$date` | Day plan |

Params and search params validated with Zod. Loaders call `queryClient.ensureQueryData(...)`.

## Folder layout

Preliminary — the structure is discussed when frontend work starts.

```
client/src/
  main.tsx
  routes/        file-based routes
  api/           http.ts (request), *.queries.ts, query-keys.ts
  features/      notes, goals, day
  i18n/          setup + en.json
  styles/        global.css (CSS variables from the prototype + base styles)
```

## Data refresh after changes

Principle: a mutation invalidates whole resources (`tasks`, `goals`, `dayPlans`, `notes`) rather than precise queries; responses are not written into the cache by hand. Logout clears the whole cache.

## Decided per stage (not now)

- Folder structure (when frontend work starts).
- Exact query keys and which mutation invalidates which resources (stage 3).
- Optimistic update for the done checkbox, if at all (stage 3).
- Porting the prototype styles into `global.css` and component modules (with the first screen).
