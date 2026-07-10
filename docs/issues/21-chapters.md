# Title

Chapters: CRUD API and timeline rail editing

## Summary

Implement manual chapters end-to-end per DESIGN.md §11.1/§5: zod-validated CRUD endpoints over
the `chapters` table, and the timeline left-rail UI — colored range spans with overlap lanes,
create via drag or button, edit/delete via modal.

## Context

Chapters are the "autobiography" differentiator (frozen requirement: manual in v1, auto-detect
deferred). They mount into the gutter issue 20 reserved.

## Scope

- `src/db/queries/chapters.ts`, `src/server/routes/chapters.ts`;
  `src/web/pages/Timeline/ChaptersRail.tsx`, `ChapterModal.tsx`;
  `i18n/{en,ja}/timeline.json` additions.

## Detailed Requirements

1. Queries: `listChapters` (order starts_on ASC, id ASC), `createChapter`, `updateChapter`
   (partial), `deleteChapter`. Timestamps `created_at/updated_at` set server-side (UTC now).
2. Routes (§9): `GET /api/chapters` → `{ chapters: [...] }`;
   `POST /api/chapters` (201, returns row); `PATCH /api/chapters/:id`;
   `DELETE /api/chapters/:id` → 204.
   Validation (shared zod schema): title 1-80 chars (trimmed, non-empty); `starts_on`/
   `ends_on` = `YYYY-MM-DD` valid calendar dates; `ends_on` null or >= `starts_on`
   (400 `ERR_VALIDATION` with field-level details); `color` in the §5 palette enum;
   `note` ≤ 2000. PATCH semantics (normative): load the existing row, merge the partial
   payload, validate the MERGED entity (so `ends_on >= starts_on` holds across the merge);
   unknown fields rejected; empty patch → 400 `ERR_VALIDATION`. Unknown id → 404. No
   overlap restriction (overlaps are legal and lane-rendered).
3. Rail rendering (SVG column in the 96px gutter, aligned to month-row geometry):
   - Span geometry: a chapter covers month rows intersecting [starts_on, ends_on ?? today]
     — clamp to the visible range; sub-month precision maps proportionally within the month
     row's pixel height.
   - Overlaps: greedy lane assignment (sort by starts_on; first free lane of max 3); >3
     concurrent → 4th+ merge into a `+n` chip at the top of the overlap region, clicking it
     lists those chapters in a popover.
   - Labels: vertical-friendly horizontal text, truncate 24 chars w/ tooltip. `auto` color
     rule (single, normative): the CLIENT resolves `auto` before POST to the least-used of
     the 9 fixed palette colors (tie → palette order) and sends the concrete color; stored
     rows therefore never keep `auto`, except the DB default on rows created outside the UI,
     which render gray.
   - Interactions: click span → modal (edit); "Add chapter" button (top of rail) → modal
     (blank); drag vertically on empty rail area → prefills starts_on/ends_on months (first
     day of start month, last day of end month); Escape cancels drag.
4. Modal: fields title, start date, end date (nullable "ongoing" checkbox), color swatches,
   note textarea; inline field errors from 400 details; delete button with confirm inside
   modal; optimistic update with rollback on API error (toast).
5. Timeline data flow: chapters fetched once per page load + after mutations; rail re-renders
   without refetching timeline stats.

## Acceptance Criteria

- [ ] API contract tests: CRUD happy paths; each validation rule → 400 with field detail;
      ends<starts rejected incl. via partial PATCH of a single date field (merged
      validation); empty patch → 400; 404s; auth 401.
- [ ] Defense-in-depth: zod rejects before SQLite; direct-DB inserts violating the ACTUAL
      §5 CHECKs (`title` length, `note` length) fail at the DB layer. (Date order and color
      enum are zod-only — §5 has no such CHECKs.)
- [ ] T7: hostile `title`/`note` strings (`<img onerror=...>`, `<script>`) render as inert
      text in the SVG rail, tooltip, popover, and modal (DOM assertions; no
      `dangerouslySetInnerHTML`).
- [ ] Rail component tests (fixture: 4 chapters incl. 2-overlap, 4-overlap, ongoing):
      lane assignment stable/deterministic, `+n` chip content, ongoing extends to today,
      clamp to visible range.
- [ ] Drag-create prefill months verified (pointer-event simulation).
- [ ] Optimistic rollback on mocked 500.
- [ ] en/ja coverage; keyboard path: rail spans focusable, Enter opens modal.

## Validation

`npm test`, lint, typecheck; manual GIF or screenshots (create-by-drag, overlap lanes) in PR.

## Dependencies

04 (table exists), 06 (routes), 17 (shell), 20 (row geometry + gutter).

## Non-goals

Auto-detected chapters (v2), chapters on other views (v2), export/import of chapters.

## Design References

DESIGN.md §11.1 (rail spec), §5 (chapters DDL/palette), §9 (endpoints), §2.2 (auto-detect
non-goal).
