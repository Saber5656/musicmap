# Title

SPA shell: routing, layout, theme, i18n (en/ja), API client, shared states

## Summary

Build the React application shell per DESIGN.md §10: router with all page routes (feature
pages as placeholders), left-nav layout, light/dark theme tokens, i18next with en/ja resource
bundles, the tokenized API client with 401 handling, the shared date-range picker, and
loading/error/empty primitives.

## Context

All view issues (18-24, 28) plug pages into this shell and MUST reuse its primitives (API
client, states, range picker, i18n). Getting the token handshake and i18n discipline right
here prevents systemic rework.

## Scope

- `src/web/`: `main.tsx`, `app/router.tsx`, `app/Layout.tsx`, `api/client.ts`,
  `i18n/` (setup + `{en,ja}/{common,errors}.json` full; feature namespaces stubbed),
  `styles/tokens.css`, `components/` (`AsyncBoundary`, `EmptyState`, `ErrorState`,
  `Skeleton`, `DateRangePicker`, `MetricToggle`), hooks (`useApi`, `useResizeObserver`).

## Detailed Requirements

1. Token handshake (§10.2, ADR-004): on boot read `location.hash` `#token=<t>` →
   `sessionStorage.setItem('mm.token', t)` → strip via `history.replaceState` → validate
   with `GET /api/auth-check` (issue 06; 204 = ok, 401 = TokenGate). `apiFetch` attaches
   `Authorization: Bearer`; a 401 anywhere renders the full-screen TokenGate page (i18n'd
   instructions to copy the URL from the terminal). No token in `localStorage`, no cookies,
   never logged.
2. `apiFetch<T>(path, opts)`: JSON by default; parses error envelope → throws
   `ApiError { code, details, status }` (the envelope's `message` is intentionally dropped:
   the client localizes by `code`); multipart pass-through for uploads (18). One retry on
   network failure for idempotent GETs only.
3. Router (react-router v7, `createBrowserRouter`): routes `/`, `/import`, `/timeline`,
   `/trends`, `/wrapped`, `/wrapped/:year`, `/map`, `/settings` — each a lazy-loaded page
   module; placeholders render the page title (i18n) + `EmptyState`. SPA fallback works with
   issue 06's server (deep-link reload).
4. Layout: left nav (7 entries, icons via inline SVG components — no icon dependency),
   active-route highlight, app version in footer (from `/api/health` once, cached);
   content region with max-width 1400px; top bar slot where pages mount `DateRangePicker`/
   `MetricToggle`.
5. Theme (§10.2): `tokens.css` defines the full custom-property set for BOTH themes
   (`:root[data-theme=light|dark]`): background/surface/text/border scales, 10 categorical
   chart colors (must pass 3:1 contrast against both surface colors — document chosen
   hexes), spacing/radius/font tokens. Pre-issue-24 persistence rule (normative): this
   issue must NOT call `/api/settings`; theme and locale overrides persist in
   `localStorage` keys `mm.theme` / `mm.locale` ONLY until issue 24, which migrates both
   once and removes them. `auto` theme follows `matchMedia('(prefers-color-scheme)')`.
6. i18n (§10.3): i18next initialized synchronously with bundled resources (no lazy fetch);
   `locale: 'auto'` resolution rule; `common.json` (nav, app chrome, metric labels, range
   presets) and `errors.json` (EVERY code from `src/core/errors.ts` — en and ja both;
   missing-key dev warning enabled). Add eslint rule `i18next/no-literal-string` via
   `eslint-plugin-i18next` (devDep — allowlisted in DESIGN §4) scoped to `src/web/**`
   excluding tests.
7. `DateRangePicker`: presets All / Last year / Last 5 years / Custom (two `YYYY-MM` month
   inputs); emits `{ from?, to? }`; URL-synced via search params (`?from=&to=`) so views are
   deep-linkable (§11.1). `MetricToggle`: plays|minutes, URL-synced `?metric=`.
8. `AsyncBoundary` wraps data loads: skeleton → error (localized by code + retry button) →
   empty (icon + message + optional CTA) — the three states every page must use.
9. Accessibility base: landmark roles, focus outline tokens, `prefers-reduced-motion`
   respected globally (disable transitions).

## Acceptance Criteria

- [ ] `npm run dev`: hash token stored+stripped; API calls carry the header (network tab /
      test); wrong token → TokenGate page in en and ja.
- [ ] All 8 route patterns render placeholders (left nav shows 7 entries — `/wrapped/:year`
      is covered by the Wrapped entry); deep-link reload works via server fallback; lazy
      chunks split (build output shows per-page chunks).
- [ ] T7 discipline: no `dangerouslySetInnerHTML` and no HTML `style` props anywhere in
      `src/web/**` (lint green — inline style attributes would violate the §13.3 CSP);
      dynamic geometry uses SVG attributes, theming uses classes + custom properties defined
      in CSS files.
- [ ] Theme toggle switches `data-theme` and persists across reload; `auto` follows OS.
- [ ] Locale auto-resolution: `navigator.language` ja → ja UI; en otherwise; manual override
      persists (client-side until 24).
- [ ] `errors.json` covers every ErrorCode (unit test iterates the enum against both bundles).
- [ ] Literal-string lint: a hardcoded JSX string fails `npm run lint` (prove once, remove).
- [ ] DateRangePicker/MetricToggle round-trip URL params (component tests with router memory).
- [ ] Vitest component smoke tests (happy render per shared component) using
      `@testing-library/react` (devDep allowed for web tests) + jsdom.

## Validation

`npm test`, `npm run lint`, `npm run typecheck`, `npm run build` (chunks), manual dev-mode
check of token flow described in PR.

## Dependencies

01-project-scaffold, 06-server-security-bootstrap.

## Non-goals

Feature page content (18-28), settings persistence server-side (24), charts (d3 arrives with
20/22/28), mobile layouts (v1 non-goal; must not break at 1024px only).

## Design References

DESIGN.md §10.1-10.3 (normative), §11 shared interaction rules, §13.3 (no inline
scripts/styles); ADR-004 (token flow).
