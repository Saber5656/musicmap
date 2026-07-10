# Title

Fastify app factory with the localhost security model

## Summary

Implement the HTTP server foundation per ADR-004 and DESIGN.md §13: `buildApp()` factory with
bearer-token auth, Host-header allowlist, security headers/CSP, unified error envelope, static
SPA serving, and loopback-only listening. No feature routes yet (only `/api/health`).

## Context

Every API issue plugs routes into this factory. The security controls here are mandatory
acceptance criteria for threats T1/T2/T3/T7/T11/T14 — they must be built first and tested
independently so later routes inherit them automatically.

## Scope

- `src/server/app.ts` (factory), `src/server/index.ts` (bootstrap), `src/server/security/`
  (`token.ts`, `hostAllowlist.ts`, `headers.ts`), `src/server/errors.ts` (envelope mapping).
- Add dependency `@fastify/static` usage (installed in 01).

## Detailed Requirements

1. `buildApp(opts): FastifyInstance` where
   `opts = { logger, token: string, port: number, staticDir?: string, version: string,
   readOnly?: boolean }`. Decorations defined HERE (with Fastify type augmentation in
   `src/server/types.d.ts`), set from opts: `mmToken: string`, `mmVersion: string`,
   `mmPort: number`, `mmReadOnly: boolean` (default false). Reserved decoration names later
   issues register (declare in the augmentation now as optional): `mmDb`, `mmDataDir`,
   `mmImportService`, `mmEnrichmentRunner`, `mmOnSettingChanged`. Later issues must use
   these names — no ad-hoc globals.
2. Token auth (`security/token.ts`):
   - `generateToken(): string` = 32 random bytes → base64url (`node:crypto randomBytes`).
   - `onRequest` hook for `/api/*` except exact `GET /api/health`: require header
     `authorization: Bearer <token>`; compare with `crypto.timingSafeEqual` on equal-length
     buffers (length mismatch → immediate 401); failure → 401 with the FULL DESIGN §9
     envelope: `{ error: { code: 'ERR_UNAUTHORIZED', message: 'Missing or invalid token',
     details: {} } }`.
3. Host allowlist (`security/hostAllowlist.ts`):
   - Global `onRequest` hook (ALL routes incl. static): allowed `host` values exactly
     `127.0.0.1:<port>` and `localhost:<port>` (compare full host:port string, case-insensitive
     hostname). Anything else, or missing Host → 403 with the full envelope
     (`code: 'ERR_FORBIDDEN_HOST'`).
4. Headers (`security/headers.ts`), `onSend` hook, every response — exact values (verbatim
   from DESIGN §13.3, restated here against transcription drift):
   - `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self';
     img-src 'self' data: blob:; connect-src 'self'; object-src 'none'; frame-ancestors
     'none'; base-uri 'none'; form-action 'self'` (`blob:` is for token-fetched artwork
     object URLs — §13.3)
   - `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`,
     `Cross-Origin-Opener-Policy: same-origin`, `Cross-Origin-Resource-Policy: same-origin`.
   No `Access-Control-Allow-*` headers anywhere; OPTIONS preflights are not special-cased
   (they fail auth like any request).
5. Error envelope (`server/errors.ts`): `app.setErrorHandler` mapping `AppError` → its HTTP
   status via a code→status table (§9: validation 400, unauthorized 401, forbidden host 403,
   not found 404, conflict/duplicate 409, too large 413, enrichment disabled 409, others 500);
   zod errors → 400 `ERR_VALIDATION` with issue list in `details`; unexpected errors → 500
   `ERR_INTERNAL` with NO stack/message leakage (log full error server-side instead).
   `setNotFoundHandler`: `/api/*` → 404 envelope; non-API GET → serve `index.html` (SPA
   fallback) when `staticDir` set, else 404.
6. Static serving: `@fastify/static` at root serving `staticDir` (`dist/web`), immutable cache
   for hashed assets (`/assets/*` → `max-age=31536000, immutable`), `index.html` no-store.
7. `listenLoopback(app, port: number, opts: { explicit: boolean }): Promise<number>`
   (exported from `server/index.ts`): `listen({ host: '127.0.0.1', port })`; on
   `EADDRINUSE` only — when `explicit` is false, try `port+1..port+9` sequentially and
   return the bound port; when `explicit` is true, throw a clear error (DESIGN §14).
   `explicit` comes from issue 03's `sources.port` (`'cli' | 'env'`). The host argument is
   hardcoded loopback; there is NO option to change the bind host anywhere (grep-able
   absence). Full serve orchestration (config resolution, data dirs, migrate, lock,
   recovery, stdout URL line, browser opening) is issue 07's — NOT here; the scaffold
   `startServer()` from issue 01 is reduced to a thin dev-only wrapper over
   `buildApp` + `listenLoopback`.
8. `GET /api/health` → `200 { ok: true, version }` (no auth, but Host check applies).
9. `GET /api/auth-check` → `204` empty body, token-protected — the SPA boot handshake
   target (DESIGN §9/§10.2), so the shell (issue 17) never depends on a feature route.

## Acceptance Criteria

- [ ] fastify-inject tests: `/api/health` 200 without token; any other `/api` 401 without /
      with wrong / with truncated token; `/api/auth-check` 204 with correct token, 401
      otherwise.
- [ ] Host tests: `Host: 127.0.0.1:<port>` and `localhost:<port>` pass; `evil.com`,
      `127.0.0.1:9999` (wrong port), `[::1]:<port>`, and missing Host → 403 — including for
      `/` static and `/api/health`.
- [ ] Header assertions: every §13.3 header present with exact values on `/api/health`, a 404,
      and a static asset response.
- [ ] Error envelope: thrown `AppError('ERR_CONFLICT')` → 409 with the full
      `{ error: { code, message, details } }` envelope; generic `Error` → 500 `ERR_INTERNAL`
      and response contains neither its message nor stack; 401/403 responses carry the full
      envelope too.
- [ ] `OPTIONS /api/imports` without auth → 401 (no CORS special-casing).
- [ ] Timing-safe compare used (code review). Log-leak test: using the issue-03 logger with
      captured output, run one failing and one succeeding auth request — captured logs
      contain neither the token value, any substring of it (≥ 8 chars), nor the string
      `Bearer ` with a value.
- [ ] Static cache assertions: `/assets/<hashed>.js` → `Cache-Control: public,
      max-age=31536000, immutable`; `/` and `/timeline` (SPA fallback) → `Cache-Control:
      no-store`.
- [ ] `listenLoopback`: fallback binds port+1 when busy (explicit=false); explicit=true →
      throws; bound host asserted to be `127.0.0.1`; repo-wide grep shows no
      `0.0.0.0`/`::`/host option.
- [ ] SPA fallback returns `index.html` for `/timeline` when staticDir set.

## Validation

`npm test` (suite `src/server/**/*.test.ts` using `app.inject`, no real sockets except one
listen test that binds an ephemeral port and asserts the stdout line format), lint, typecheck.

## Dependencies

01-project-scaffold, 03-config-and-paths.

## Non-goals

Feature routes (14, 19-28), CLI UX incl. browser opening (07), multipart limits (14), CSRF
docs (31 verifies), rate limiting (not needed on loopback).

## Design References

ADR-004 (normative); DESIGN.md §9 (envelope/status), §13.2 T1-T3/T7/T11/T14, §13.3, §14 (port
busy, instance lock is issue 07).
