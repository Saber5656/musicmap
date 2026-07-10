# Title

Import wizard and imports management UI

## Summary

Build the `/import` page per DESIGN.md §10.1: a source-picker wizard with per-source
"how to obtain your export" guides (en/ja), drag-and-drop upload with progress, parse-progress
polling, the import report view (totals + skip breakdown + warnings), and the imports list
with delete — all against the issue-14 API.

## Context

This is the first real user workflow and the make-or-break onboarding moment. The guides must
encode the research-doc steps (e.g. "switch Takeout history format to JSON") because users
WILL arrive with the wrong file otherwise.

## Scope

- `src/web/pages/Import/` (wizard, guides content, progress, report, list components),
  `src/web/i18n/{en,ja}/import.json`.

## Detailed Requirements

1. Wizard steps: (1) pick source card (Spotify / Apple Music / YouTube Music, logo-less text
   cards; POST enum values `spotify` / `apple_music` / `youtube_music` per issue 14) →
   (2) guide + dropzone → (3) progress → (4) report.
2. Guides (i18n content keys, en+ja; steps normative from `docs/research/*.md`):
   - Spotify: privacy settings → Extended streaming history checkbox → wait for email → upload
     the ZIP as-is. Note: "Account data" ZIP is NOT supported (name the difference).
   - Apple: privacy.apple.com → request Apple Media Services info → upload ZIP or the
     `Apple Music Play Activity.csv`.
   - YouTube Music: takeout.google.com → YouTube and YouTube Music → **History format = JSON**
     (highlighted warning box) → upload ZIP or `watch-history.json`. When the API returns
     `ERR_NO_RECOGNIZED_FILES` with `hint: 'youtube_html_format'`, show the specific "your
     export is HTML; re-export with JSON" message.
3. Dropzone: file input + drag-drop; client-side pre-checks (extension in
   `.zip/.json/.csv`, size ≤ 4 GiB) with localized messages; single file per upload.
4. Upload progress: `XMLHttpRequest` (fetch lacks upload progress) with the bearer header;
   percent bar. Then parse progress: poll `GET /api/imports/:id` every 1 s (§6.1). Status
   `uploaded` renders a "queued — waiting for the current import" state (another import may
   be parsing, §6.1 single worker); `parsing` renders phase, files X/Y, events imported,
   bytes %; terminal states (`completed`/`failed`) stop polling. Progress display continues
   correctly after page reload (resume by import id from the list).
5. Report view (§6.6): totals row (events, duplicates, artists/tracks created, date range),
   per-file collapsible table, skipped-breakdown table with localized reason labels
   (`podcast`, `ad`, `deleted`, `non_audio`, `invalid`, `title_unparsed`, ...), warnings list
   (code + detail, monospace). Duplicate-heavy re-import (events 0 / duplicates N) gets a
   friendly "already imported — nothing new" callout, not an error tone.
6. Error states: every import-relevant `ErrorCode` from `src/core/errors.ts` — all §6.7
   codes PLUS `ERR_DUPLICATE_UPLOAD`, `ERR_CONFLICT` (read_only/parsing reasons),
   `ERR_VALIDATION`, `ERR_INTERRUPTED`, `ERR_NOT_FOUND` — renders localized actionable text.
   For `ERR_NO_RECOGNIZED_FILES` specifically: show the found entry names from
   `details` (scrollable list), the `details.hint` message when present (ytm HTML →
   JSON re-export instruction), and `didYouMean` rendering ("This looks like a {{source}}
   export — import it under that source?" with a switch-source button).
7. Imports list: newest-first cards/table (source, filename, date, status chip, totals);
   delete with typed-confirm modal ("delete import and its N events"; localized copy, but
   the required keyword is the literal `DELETE` — same convention as issue 24); 409
   read-only/parsing errors surfaced as toasts; list auto-refreshes while any import is
   `parsing`.
8. Duplicate upload (409 `ERR_DUPLICATE_UPLOAD`): dialog linking to the existing import row.
9. All strings via i18n; uses shell primitives (AsyncBoundary etc.); no new deps.

## Acceptance Criteria

- [ ] Component tests (jsdom + msw-less: stub `apiFetch`/XHR): wizard flow reaches report on
      a mocked happy path; each §6.7 error code renders its specific message; ytm HTML hint
      shows the re-export instruction.
- [ ] Progress poller stops on `completed`/`failed`; reload-resume verified (mock import in
      `parsing`).
- [ ] Delete confirm requires exact keyword; success removes row; 409s toast.
- [ ] en and ja render all wizard/guide/report strings (i18n key-coverage test: every key
      used exists in both bundles; no literal-string lint violations).
- [ ] T7 rendering test: a filename and a warning detail containing `<img onerror=...>` and
      `<script>` markup render as escaped text (no element injected — DOM assertion); no
      `dangerouslySetInnerHTML` anywhere on the page (lint already forbids; test documents).
- [ ] Queued (`uploaded`) state renders and transitions to parsing when the worker picks it
      up (mocked sequence).
- [ ] a11y: dropzone keyboard-operable; progress has `role=progressbar` with values.

## Validation

`npm test`, `npm run lint`, `npm run typecheck`; manual end-to-end against a real served
instance with the spotify mini fixture (screenshot in PR).

## Dependencies

14-import-api-and-cli, 17-web-shell-i18n.

## Non-goals

Server changes (any gap found → raise on issue 14), E2E automation (30), guides with
screenshots (text-only v1, DESIGN §2.2).

## Design References

DESIGN.md §10.1 (/import), §6.1/§6.6/§6.7 (contract), §14 (wrong-source row);
docs/research/*.md (guide steps).
