# Title

Performance validation against DESIGN budgets (1M-event fixture)

## Summary

Implement `npm run perf`: a repeatable measurement harness that runs every DESIGN.md §15
budget against the `perf`-scale synthetic fixture (issue 29), reports a comparison table, and
fixes any misses found (indexes/queries/batching) — plus an optional nightly CI report job.

## Context

Budgets exist to keep the app usable at real long-history scale. This issue turns them from
prose into a script whose output is pasted into releases; it is diagnostic, not a per-PR
gate (nightly + manual).

## Scope

- `scripts/perf.ts`; `package.json` script; scheduled non-blocking perf workflow;
  query/index fixes as needed.

## Detailed Requirements

1. Harness (`scripts/perf.ts`). Isolation is mandatory: the harness creates a temp
   `MUSICMAP_DATA_DIR` (mkdtemp), cleans it up afterwards, and asserts the user's default
   data dir is never touched (path check before/after).
   - Steps, each timed with `performance.now()` and peak-RSS sampled via
     `process.memoryUsage().rss` polling (500 ms):
     1. Generate (or reuse cached) perf fixtures (seed 42).
     2. Import Spotify perf ZIP via the framework directly (no HTTP) → events/s, total
        time.
     3. Import Apple + YTM perf files (same metrics).
     4. RSS budget step (matches §15's "500 MiB upload import" wording): pad the Spotify
        perf ZIP with pre-compressed filler entries to ≥ 500 MiB, run it through the REAL
        upload path (multipart POST against the booted app from step 6's instance or
        `app.inject` streaming) → peak RSS ≤ 1.5 GiB.
     5. `rebuildAggregates` cold → time.
     6. API latency: boot the real app; 50 warm calls each with EXACT query strings:
        `/api/timeline?bucket=month&from=<earliest>&to=<latest>&topN=10&metric=plays`
        (full 12y range), `/api/trends?mode=artist&topN=10&metric=plays`,
        `/api/trends?mode=genre&topN=10&metric=plays`, `/api/wrapped/<busiest>` where
        busiest = the year with max plays from `getWrappedYears` data,
        `/api/genre-map?metric=plays` (seed `artist_genres` synthetically first: 3 genres
        × top 5k artists) → p50/p95 per endpoint.
     7. SPA cold-load: Playwright measured `domInteractive` on the built app (3 runs,
        median).
     8. Genre-map layout: run issue 28's `computeLayout` on the 150-node payload → time.
   - Output: markdown table (budget | measured | pass/fail) to stdout AND
     `perf-report.md`; process exit code 0 even on budget misses (report-only), but a
     `--strict` flag exits 1 on any miss (used manually/release).
2. Budgets encoded as constants imported from ONE module (`scripts/perfBudgets.ts`)
   mirroring §15 exactly — the table in DESIGN stays the source of truth (comment linkage).
3. Fix pass: if any §15 budget misses on the reference machine (document specs used),
   optimize within existing design: candidates pre-identified — composite index
   `(track_id, played_at)` if timeline topTracks lags; consolidating the timeline
   duration-coverage/topTracks probes (issue 15's single-GROUP-BY note);
   prepared-statement reuse in rebuild loop; `PRAGMA mmap_size`. Schema changes require a
   new migration (002_perf_indexes.sql) — allowed by ADR-006 discipline. Design-level
   changes (e.g. new cache tables) are OUT of scope → escalate as a design question
   instead.
4. Scheduled CI job (required; name `perf-nightly`): triggers = `schedule:` cron (weekly
   minimum) AND `workflow_dispatch` (for the acceptance run). Runs `medium` scale only (CI
   runners are weak; 1M is local/release practice), uploads `perf-report.md` artifact,
   never blocks PRs. Hardening identical to issue 02: every `uses:` SHA-pinned, top-level
   `permissions: contents: read`, no secrets referenced.

## Acceptance Criteria

- [ ] `npm run perf` completes end-to-end and emits the full budget table.
- [ ] All §15 budgets PASS on the reference machine at perf scale (paste `perf-report.md`
      in the PR; machine specs stated). Any unfixable-within-scope miss → documented
      escalation issue linked, budget table shows it, and the user-visible impact is
      described.
- [ ] Import throughput measured ≥ 8,000 events/s (budget) with the batching from §6.8
      unchanged or improved.
- [ ] Perf workflow runs green on `medium` scale via `workflow_dispatch` (link the run);
      workflow is SHA-pinned with `contents: read` permissions.
- [ ] RSS step provably runs through the upload path with a ≥ 500 MiB archive; peak RSS
      reported against the 1.5 GiB budget.
- [ ] Data-dir isolation asserted (user default dir untouched; temp dir cleaned up).
- [ ] Any added migration/index has its own micro-benchmark note (before/after) in the PR.
- [ ] Harness itself is deterministic re: fixtures (seed pinned) and warns when thermal/CI
      variance exceeds 20% between two consecutive runs (runs twice, compares).

## Validation

Local run at perf scale + CI `workflow_dispatch` at medium scale; both outputs in PR.

## Dependencies

14 (import/upload path), 15, 16 (queries), 28 (`computeLayout` for the layout step),
29 (perf fixtures).

## Non-goals

Load/concurrency testing (single-user app), micro-benchmark framework adoption, changing §15
budgets (design change process), optimizing beyond budget compliance.

## Design References

DESIGN.md §15 (normative budgets), §6.8, §8.4; ISSUE_PLAN §6.5; ADR-006 (migration
discipline).
