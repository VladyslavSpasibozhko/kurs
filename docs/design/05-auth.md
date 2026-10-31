# 05 — Authentication

Status: ✅ approved (2026-10-31) — email + password; Google login comes later

Every decision below has its reason in `00-decisions.md` → "Authentication".

## Summary

- Email + password, open sign-up; register logs the user in.
- Passwords hashed with `scrypt` from `node:crypto` (no dependency).
- Session: access JWT (15 min) + refresh token (30 days, sliding) with rotation and reuse detection, both in HttpOnly cookies.
- JWTs signed with `hono/jwt` (HS256, `JWT_SECRET`).
- Not now (see "Later"): Google login, email verification, password reset by email, rate limiting.

## Routes

All JSON, defined in `@kurs/http/routes/auth.ts`. Inputs in `@kurs/shared/inputs`.

| Route | Body | Success | Errors |
|---|---|---|---|
| `POST /auth/register` | `RegisterInput { email, password }` | 201 `{ userId }` + cookies | `VALIDATION`, `CONFLICT` (email taken) |
| `POST /auth/login` | `LoginInput { email, password }` | 200 `{ userId }` + cookies | `VALIDATION`, `INVALID_CREDENTIALS` |
| `POST /auth/refresh` | — (refresh cookie) | 200 `{ userId }` + new cookies | `UNAUTHORIZED` |
| `POST /auth/logout` | — | 204, cookies cleared | — (always succeeds) |
| 🔒 `GET /auth/session` | — | 200 `{ userId }` | `UNAUTHORIZED` |
| 🔒 `PATCH /auth/password` | `ChangePasswordInput { currentPassword, newPassword }` | 204 | `VALIDATION`, `INVALID_CREDENTIALS` |

- `login` and `refresh` set `status: 200` explicitly (not the POST default 201).
- `logout` is not 🔒: it must work with an expired access token.

## Schemas (`@kurs/shared`)

```ts
EmailSchema         = z.string().trim().toLowerCase().pipe(z.email())
PasswordSchema      = z.string().min(8).max(128)        // no character-mix rules, no trimming

RegisterInput       = { email: EmailSchema, password: PasswordSchema }
LoginInput          = { email: EmailSchema, password: z.string().min(1) }   // old passwords keep working if the rule tightens
ChangePasswordInput = { currentPassword: z.string().min(1), newPassword: PasswordSchema }
```

`ErrorCode` gets `INVALID_CREDENTIALS` (401).

## Flows

**Register:** validate → `hashPassword` → `INSERT users` (unique violation `23505` → `CONFLICT`; no check-then-insert) → issue session → 201.

**Login:** validate → find user by email →
- found: `verifyPassword(user.passwordHash, password)`;
- not found: `verifyPassword(DUMMY_HASH, password)` (same timing), result ignored;
- fail → `401 INVALID_CREDENTIALS` ("Wrong email or password", same for both cases); ok → issue session → 200.

**Issue session:** random 32-byte refresh token → `INSERT refresh_tokens { tokenHash: sha256(token), userId, used: false, expiresAt: now + 30d }` → set both cookies.

**Refresh:** read refresh cookie → find by `sha256(token)` →
- not found or expired → 401;
- `used = true` → reuse detected → mark all of the user's tokens used → 401;
- ok → mark it used → issue a new session (new 30 days) → 200.

**Logout:** mark the current refresh token used (if any) → clear both cookies → 204.

**Change password:** verify `currentPassword` (fail → `INVALID_CREDENTIALS`) → store `hashPassword(newPassword)` → 204. Sessions are not touched (all devices stay logged in).

**`requireAuth`:** verify access JWT (signature + `exp`) → `userId` in context. No DB lookup. Missing/invalid → 401.

## Password hashing (`server/src/auth/password.ts`)

```ts
export const SCRYPT_CONFIG = {
  keyLen: 64, saltLen: 16,
  N: 2 ** 16, r: 8, p: 2,            // OWASP option, ~64 MiB per hash
  maxmem: 128 * 1024 * 1024,         // Node's default limit (32 MiB) is too low
} as const;
```

Stored in `users.passwordHash` as one string:

```
scrypt$N=65536,r=8,p=2$<salt base64>$<hash base64>
```

- `hashPassword`: new random salt, settings from `SCRYPT_CONFIG`.
- `verifyPassword`: N / r / p / salt from the **stored string** (so raising settings later never breaks old hashes); key length = stored hash length; only `maxmem` from config; compare with `timingSafeEqual`.
- Async `scrypt` only (runs on the thread pool, does not block other requests).
- Settings are a code constant, not `.env`.

## Cookies

| Cookie | Content | Flags | Lifetime |
|---|---|---|---|
| `access` | JWT `{ sub: userId, exp }` | HttpOnly, SameSite=Lax, Secure (prod), `Path=/api` | 15 min (`ACCESS_TTL`) |
| `refresh` | 32 random bytes, base64url; DB keeps only SHA-256 | HttpOnly, SameSite=Lax, Secure (prod), `Path=/api/auth` | 30 days, sliding (`REFRESH_TTL`) |

- Refresh token hashed with SHA-256, not scrypt: it is random, there is nothing to guess.
- CSRF: no tokens. `SameSite=Lax` blocks cookies on cross-site non-GET requests; JSON requests need a CORS preflight that only `CLIENT_URL` passes.
- `JWT_SECRET` in the `.env` Zod schema, at least 32 characters.
- An access JWT stays valid until it expires (up to 15 min) after logout or password change — it is not checked against the DB.

## Client

- `request()` keeps one shared in-flight refresh promise: every 401 waits for it, then retries once; refresh failed → login screen.
- Accepted: two tabs refreshing at the same moment can trigger reuse detection and log the user out. Possible later: Web Locks (`navigator.locks`).
- Pages: `/login`, `/register`, account page with the change-password form.

## Refresh tokens are never deleted

No cleanup. Used rows must stay until they expire anyway (reuse detection). A one-off `DELETE … WHERE "expiresAt" < now()` is safe at any time.

## Manual checklist (until tests exist)

- Register → in the app; logout → login screen; login again.
- Register with the same email (any case, with spaces) → "email already registered".
- Wrong password and unknown email → same message.
- Access token expiry → transparent refresh; Goals page (two parallel requests) does not log out.
- Reusing an old refresh token → all sessions revoked.
- Change password → old password fails, new one works.
- User A cannot read or change user B's notes, goals, tasks, day plans (expect 404).

## Later

- Google login (`openid-client` v6, Authorization Code + PKCE): migration adds `users.googleId text UNIQUE` (nullable) and makes `passwordHash` nullable. Never auto-link a Google account to an existing account by email (emails are not verified).
- Email verification; password reset by email (same email service and token table).
- Rate limiting on login/register — required before public deployment.
