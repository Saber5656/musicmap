# ADR-004: Localhost security model (loopback bind + per-session token + Host allowlist)

- Status: Accepted (2026-07-10)
- Deciders: Fable (design), per frozen security posture

## Context

A localhost web app is not automatically safe: any website open in the user's browser can send
requests to `http://127.0.0.1:<port>` (CSRF-style), and DNS-rebinding lets a remote page bypass
same-origin checks against localhost servers. The API can trigger imports, delete data, and
toggle outbound network access, so it must not be callable by arbitrary web pages. The app is
OSS and will be run by non-expert users; secure defaults matter more than configurability.

## Decision

1. The server binds **only** to `127.0.0.1` (IPv4 loopback). There is **no** `--host` option in
   v1 — LAN exposure is a non-goal, removed at the option level, not just defaulted off.
2. On startup the server generates a random 256-bit **session token**. The CLI opens
   `http://127.0.0.1:<port>/#token=<token>`; the SPA moves the token from the URL fragment to
   `sessionStorage` (fragments are not sent over HTTP nor stored in server logs) and sends it as
   `Authorization: Bearer <token>` on every `/api` request. All `/api` routes except
   `GET /api/health` return `401` without it. Constant-time comparison.
3. **Host-header allowlist**: requests whose `Host` is not `127.0.0.1:<port>` or
   `localhost:<port>` are rejected with `403` (DNS-rebinding defense), including static assets.
4. No cookies, no CORS headers (same-origin SPA), `X-Content-Type-Options: nosniff`,
   `Referrer-Policy: no-referrer`, and a strict CSP
   (`default-src 'self'; connect-src 'self'; img-src 'self' data: blob:; object-src 'none';
   frame-ancestors 'none'; base-uri 'none'` — `blob:` solely for token-fetched artwork
   object URLs; full string in DESIGN §13.3).
5. Token rotates every server start; it is never written to disk.

## Consequences

- Malicious web pages cannot exploit the API: they lack the token (CSRF) and fail the Host check
  (DNS rebinding); loopback bind stops LAN attackers entirely.
- Losing the URL token mid-session (e.g. opening a fresh tab manually) requires re-copying the
  URL from the terminal — acceptable; the CLI prints the full tokenized URL and `musicmap serve`
  re-prints it on demand.
- Multiple simultaneous browsers/tabs work (same token); multiple server instances get distinct
  tokens and ports.

## Alternatives considered

- **No auth, rely on loopback**: defeated by CSRF/DNS-rebinding; rejected.
- **Cookie session + SameSite + CSRF tokens**: more machinery, browser-dependent behavior;
  bearer-header approach is simpler and stronger for this topology.
- **HTTPS with self-signed certs on loopback**: certificate warnings for no real gain on
  127.0.0.1; rejected for v1.
