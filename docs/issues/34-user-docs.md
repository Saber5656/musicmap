# Title

User documentation: README, PRIVACY, CONTRIBUTING, and in-app help finalization

## Summary

Write the public-facing documentation set for the v1 release: a complete README (concept,
screenshots, quickstart, per-service export guides, troubleshooting, FAQ), PRIVACY.md
(data-handling statement matching the implementation), CONTRIBUTING.md (dev setup, layering
rules, migration/i18n/security checklists), and a final consistency pass over in-app help
strings (en/ja) — verifying every documented command verbatim.

## Context

Last issue: everything it documents already exists, so docs are verified against reality, not
intentions. The privacy statement is a product feature here — it must match ADR-003/§13
behavior exactly (what leaves the machine and when).

## Scope

- `README.md` (rewrite), `PRIVACY.md`, `CONTRIBUTING.md`, `docs/screenshots/` (6-8 PNGs),
  in-app i18n copy audit; link SECURITY.md (issue 31) and CHANGELOG (33).

## Detailed Requirements

1. README structure (en; a short ja summary section at top linking full en docs):
   1. Hero: one-paragraph concept (再生履歴から音楽の自分史/"your listening
      autobiography"), 2 screenshots (timeline+chapters, genre map).
   2. Features table (4 visualizations + chapters + i18n + local-first).
   3. Privacy promise box: local-only by default; opt-in enrichment sends only
      artist/album names to MusicBrainz/CAA; link PRIVACY.md.
   4. Quickstart: requirements (Node >= 24), install-from-source commands (verbatim from
      33's README section), first import walkthrough.
   5. Getting your data: per-service step-by-step (condensed from research docs, matching
      the in-app guides; includes wait-time expectations and the Takeout JSON-format
      warning).
   6. Troubleshooting: wrong-source error, HTML Takeout, port busy, Node version,
      better-sqlite3 build toolchain (K4), where logs/data live (§17 per-OS table),
      how to reset (delete data dir), lost token URL (restart serve / re-copy).
   7. FAQ: why no Spotify login; why minutes missing for YouTube Music (honest-data rule);
      genre "unknown" share; can I use multiple accounts (yes — multiple imports merge);
      is my data uploaded anywhere (no; enrichment caveat).
   8. Roadmap (v2 items from ISSUE_PLAN §7), Contributing/Security/License links, CI badge.
2. PRIVACY.md (en + ja both, full text) — exact transmission statement (normative wording
   direction): "No listening history, timestamps, play counts, raw export files, tokens,
   or logs are ever transmitted. With enrichment OFF (the default), musicmap makes no
   network requests at all. If you enable enrichment, artist names and selected album
   titles are sent to MusicBrainz, and release-group identifiers are sent to Cover Art
   Archive / archive.org to fetch artwork." Plus: data stored (categories + location),
   caching/retention, deletion path, raw-file retention + PII note (files you import may
   contain IPs; the database drops them, raw files remain locally until you delete the
   import — mirrors §6.4 and issue 24's delete flows), logs (local only), no
   telemetry/analytics of any kind.
   Every claim cross-checked against code/tests (list the verifying test names in the PR
   description, not the doc).
3. CONTRIBUTING.md: dev setup (`npm ci`, `npm run dev`, test commands), repo layering rules
   (§3.2 + lint enforcement), how to add a migration (ADR-006 checklist), i18n rules (both
   bundles or CI fails), PR security checklist mapping changed areas to ALL relevant §13.2
   rows — parsers/import (T4-T6, T10), server/routes (T1-T3, T8, T11), web/UI (T7, T11,
   T13), enrichment (T9, T13), logging (T14), dependencies/workflows/release (T12, §13.4),
   data-at-rest (T10) — naming which tests to extend per area; dependency policy (§4
   allowlist + escalation), fixture regeneration, release process pointer (33).
4. In-app copy audit: sweep all `i18n/{en,ja}` bundles for consistency with final behavior
   (error messages actionable, guide steps match README wording); fix drift; ja review pass
   for natural phrasing (not machine-literal).
5. Screenshots: dedicated script `npm run docs:screenshots` (new, this issue — NOT part of
   the issue-30 E2E suite, which excludes visual assets): boots a temp instance, imports
   the medium fixture, seeds enrichment via the issue-25 test seam + mock, then captures
   `/`, `/import`, `/timeline` (with chapters), `/trends`, `/wrapped/<busiest>`, `/map`
   into `docs/screenshots/` — light theme, en, 1440×900; committed PNGs ≤ 300 KB each
   (compress). README hero embeds 2 of the 6 committed shots.
6. Verbatim verification: every command block in README/CONTRIBUTING executed on a clean
   checkout (script `scripts/verify-docs-commands.sh` optional but the manual log is
   required in the PR).

## Acceptance Criteria

- [ ] README sections 1-8 complete; all commands verified verbatim (log in PR); screenshots
      current and reproducible.
- [ ] PRIVACY.md en+ja complete; each claim mapped to a verifying test/implementation in
      the PR description; no claim contradicts §13/ADR-003 behavior.
- [ ] CONTRIBUTING.md covers all listed topics; a newcomer following it reaches green
      `npm test` (verified on clean clone).
- [ ] i18n audit: key-coverage tests still green; drift fixes listed; ja copy reviewed.
- [ ] Cross-links resolve (README ↔ PRIVACY ↔ SECURITY ↔ CONTRIBUTING ↔ CHANGELOG); CI
      badge live.
- [ ] ISSUE_PLAN §6.7 docs gate satisfied and noted in the PR.

## Validation

Clean-checkout command log, link check (manual or `lychee` local run), rendered-markdown
review screenshots in PR.

## Dependencies

30 (features frozen/stable), 31 (SECURITY.md exists to link; checklist content), 33
(install story final). Runs after 32 when referencing performance evidence. Content
sources: all research docs, DESIGN §17, ISSUE_PLAN §7.

## Non-goals

Docs website (GitHub README is v1), video/GIF tours, translated full README (ja summary
only), API reference docs (internal API, not public).

## Design References

DESIGN.md §1, §2, §13.4, §17; ADR-003 (privacy claims); docs/research/*.md (guide accuracy);
ISSUE_PLAN §6.7, §7.
