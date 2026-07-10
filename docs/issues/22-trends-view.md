# Title

Trends streamgraph view and `/api/trends` route

## Summary

Implement the trends route (thin over issue 16's `getTrends`) and the `/trends` page per
DESIGN.md §11.2: d3 streamgraph (wiggle offset, inside-out order) with artist/genre mode
toggle, legend series toggling, hover tooltip with per-bucket values, and enrichment-gated
genre mode.

## Context

First d3-heavy surface; establishes the project's d3-in-React pattern (d3 computes geometry,
React renders paths) that the genre map (28) follows.

## Scope

- Route in `src/server/routes/stats.ts`; `src/web/pages/Trends/` (`Streamgraph.tsx` pure
  presentational, page container); add d3 packages (`d3-shape`, `d3-scale`, `d3-array`);
  `i18n/{en,ja}/trends.json`.

## Detailed Requirements

1. Route `GET /api/trends?mode&from&to&topN&metric`: zod (mode enum, topN 1-20 default 10),
   reply per §9. Genre mode passes through even when only `__unknown__` exists (UI gates).
2. Streamgraph component (props: buckets, series, metric, hiddenIds):
   - Scales: x = point scale over buckets (time labels: year ticks only — first bucket of
     each year); y = linear from stacked extents.
   - `d3.stack()` keys = visible series; `stackOffsetWiggle` + `stackOrderInsideOut`;
     ONE ordering exception (§11.2): `__other__` is excluded from inside-out ordering and
     rendered as the bottom layer. `__unknown__` participates in normal stack order.
     Fills: `__other__` = neutral gray token; `__unknown__` = hatched gray (SVG pattern
     def).
   - Area path per series (`curveBasis`); series color = categorical palette by stable index
     (sorted by total desc, stable across metric toggles within a load).
   - Hover: nearest-bucket vertical guide line; tooltip lists bucket label + each visible
     series value + % of bucket total (sorted desc, max 12 rows, "…" beyond); values
     localized (`Intl.NumberFormat`).
   - No entry animation when `prefers-reduced-motion`; otherwise 300ms path fade only.
   - Responsive to container width (shell hook), height fixed 420px. The chart never
     introduces page-level or chart-level horizontal scroll: x positions compress for long
     ranges (including > 24 years).
3. Page: DateRangePicker + MetricToggle + mode toggle (Artist | Genre). Genre-mode gating is
   SELF-CONTAINED (no dependency on overview/enrichment endpoints): the toggle is always
   clickable; on switching to Genre, if the response contains no real genre series (only
   `__unknown__`/`__other__` or empty), the chart area renders the explainer state with an
   enrichment CTA linking to `/settings#enrichment` (the single gray band renders behind
   it when present).
   Legend rules (explicit): `__other__` — not shown in the legend; `__unknown__` — shown
   with an info tooltip but its toggle is disabled; real series — click toggles visibility
   (recompute stack client-side).
4. Empty/edge states: <2 buckets in range → "range too narrow" EmptyState; all series zero →
   EmptyState.

## Acceptance Criteria

- [ ] Route contract golden test (artist + genre modes on the fixture; `__unknown__`
      fractional numbers); 401 without token (route registered inside the app-factory hook
      chain).
- [ ] Streamgraph unit tests (render to string / jsdom): correct path count, `__other__`
      pinned bottom, hidden-series recompute drops layer, hatched pattern applied.
- [ ] T7: a hostile artist name (`<img onerror=alert(1)>`) renders as inert text in legend,
      tooltip, and SVG `<text>` (DOM assertion; no dangerouslySetInnerHTML).
- [ ] Tooltip math: bucket percentages sum to 100±0.5% (test on fixture bucket).
- [ ] Genre-mode gating both ways (response with only `__unknown__` → explainer state;
      response with ≥1 real genre series → chart).
- [ ] Deterministic colors across re-renders (snapshot twice).
- [ ] Reduced-motion disables animation (class/style assertion).
- [ ] en/ja coverage; screenshots (artist + genre, light/dark) in PR.

## Validation

`npm test`, lint, typecheck; manual on mini fixture with seeded genres.

## Dependencies

16-stats-extended, 17-web-shell-i18n.

## Non-goals

Brush zoom (range picker covers it), CSV export, genre map (28), server-side rendering of
charts.

## Design References

DESIGN.md §11.2 (normative), §9 (payload), §8.1 (metrics), §10.2 (states).
