# Title

Genre map: data route and seeded force-layout view

## Summary

Implement the genre map end-to-end per DESIGN.md §11.4 and ADR-005: the `/api/genre-map`
route computing nodes (top-150 genres by weight), co-occurrence edges (top-4-per-node
pruning), per-node top artists and the unknown share; and the `/map` page with a
deterministic seeded d3-force layout, SVG pan/zoom, hover/click interactions, side panel, and
the enrichment-gated empty states.

## Context

The namesake feature. Server does data shaping (testable numbers); client does layout
(seeded, cached) and rendering. Every state where data is missing must explain itself
(consent off / job running / thin data).

## Scope

- `getGenreMapData` in `src/db/queries/stats.ts`; route in `src/server/routes/stats.ts`;
  `src/web/pages/GenreMap/` (`layout.ts` pure, `MapCanvas.tsx`, `SidePanel.tsx`, page);
  add `d3-force`, `d3-zoom`, `d3-selection`; `i18n/{en,ja}/map.json`.

## Detailed Requirements

1. `getGenreMapData(db, tz, { from?, to?, metric })` (server, normative math):
   - Per-artist weight in range from `agg_monthly_artist` (sum metric over buckets in
     range).
   - Node weight per genre: **fractional** fan-out (weight/k per artist with k genres —
     same rule as trends §11.2; keeps map/trends/wrapped consistent).
   - `unknownWeight`: summed weight of artists with zero genres; `totalWeight`: all artists.
   - Nodes: top 150 genres by weight desc (ties name ASC); the remainder is returned as an
     explicit `others: Array<{ genreId, name, weight }>` list (weight desc, full remainder —
     ADR-005 "never silently dropped": the side panel shows it, not just a count).
   - Edges: for each artist with ≥2 genres among the kept nodes, add its FULL weight (not
     fractional) to each unordered genre pair; prune to union of each node's top-4 edges by
     weight; emit `{ a: genreIdSmaller, b: genreIdLarger, weight }`.
   - Per-node topArtists: top 10 by artist weight (full weight, not fractional — panel shows
     real listening), `{ artistId, name, weight }`.
   - Full response schema (normative; extends the §9 example):
     ```ts
     { metric: Metric; range: { from: string; to: string };   // resolved echo
       totalWeight: number; unknownWeight: number;
       nodes: Array<{ genreId: number; name: string; weight: number;
                      topArtists: Array<{ artistId: number; name: string; weight: number }> }>;
       edges: Array<{ a: number; b: number; weight: number }>;
       others: Array<{ genreId: number; name: string; weight: number }> }
     ```
2. Route `GET /api/genre-map?from&to&metric`: zod — `metric` ∈ `plays|minutes`, default
   `plays`, invalid → 400 `ERR_VALIDATION`; always 200 per DESIGN §9 — never an error when
   enrichment is disabled (cached genres stay useful after disabling); the UI decides
   gating via `/api/enrichment/status` + overview.
3. Layout (`layout.ts`, pure, unit-tested): mulberry32 PRNG seeded with constant
   `0x6d6d6170`; initial positions on a seeded golden-angle spiral; forces:
   `forceLink(edges).distance(d => 40 + 200/(1+d.weight/maxW)).strength(0.4)`,
   `forceManyBody().strength(-80)`, `forceCollide(r + 2)`, `forceCenter(0,0)`;
   `simulation.stop()`, tick exactly 300 times synchronously; radius scale: sqrt weight →
   [6, 40]. Returns `{ id, x, y, r }[]` — deterministic (same input → identical output;
   overrides d3's randomness via `simulation.randomSource`).
   Cache (page-level responsibility, not layout.ts): `sessionStorage['mm.map.' +
   fnv1a(ordered (genreId, weight) list)]` per DESIGN §11.4; `computeLayout` is injected
   into the page component so tests assert it is NOT called on re-render with an unchanged
   node hash.
4. Rendering (`MapCanvas`): single `<svg>`; `<g>` transformed by d3-zoom (scaleExtent
   [0.3, 4], wheel+drag+dblclick-zoom); edges as lines (strokeOpacity 0.15-0.6 ∝ weight);
   nodes circles (palette color = categorical hash of genre name; consistent across
   sessions) + labels (visible when zoom ≥ 0.8 OR node in top-20 by weight); hover →
   tooltip (name, weight, % of totalWeight); click → SidePanel (name, weight, %, top-10
   artist list); Escape/outside-click closes. Reduced-motion: no zoom transitions.
5. Page states (priority order): (a) enrichment disabled AND zero genre rows → explainer +
   consent CTA (link `/settings`); (b) runner running AND matched < 10 → progress state
   (poll status 2 s, show counts/eta); (c) data but zero nodes in range → EmptyState (range
   hint); (d) map. Fixed chip always visible when `unknownWeight > 0`: "n% of listening has
   no genre data". When `others` is non-empty, the side panel's default (no node selected)
   view lists the beyond-top-150 genres from `others`.
6. Date-range picker + metric toggle (URL-synced); refetch on change; layout recomputes only
   when node-set hash changes (§11.4).

## Acceptance Criteria

- [ ] Query golden tests on seeded fixture: node weights (fractional), unknownWeight,
      edge pruning (construct a 6-genre case where top-4 union drops a known weakest edge),
      topArtists, `others` list (force >150 genres via synthetic inserts).
- [ ] Trends-consistency invariant: sum(node weights) + unknownWeight + sum(others weights)
      == artist-mode total for the same range (test).
- [ ] Metric validation: `metric=hours` → 400; default resolves to `plays`.
- [ ] Layout determinism: two runs → byte-identical positions; snapshot committed.
- [ ] Route 200 with enrichment disabled (contract test); 401 without token.
- [ ] T7/T8: hostile genre/artist names render as inert text in labels, tooltip, and side
      panel (DOM assertion; no dangerouslySetInnerHTML); all map SQL uses bound parameters.
- [ ] Component tests: all four page states render per mocked inputs; side panel opens with
      top artists and shows the `others` list in its default view; zoom transform applied
      (d3-zoom smoke via dispatched wheel event).
- [ ] Injected `computeLayout` is not re-invoked on re-render with unchanged node hash
      (cache hit), and is re-invoked when the range changes the node set.
- [ ] en/ja coverage; screenshots (map with data, unknown chip, dark mode) in PR.

## Validation

`npm test`, lint, typecheck; manual exploration on mini fixture + seeded genres.

## Dependencies

15-stats-core, 17-web-shell-i18n, 24-settings-and-consent, 26-enrichment-runner.

## Non-goals

Canvas fallback (>150 nodes impossible by cap), time-scrubber animation (v2), genre
hierarchy/regions (v2), MST connectivity guarantee (documented ADR-005 fallback knob only if
hairballs materialize — K2).

## Design References

DESIGN.md §11.4 (normative), §9 (payload — amended here), §8.1; ADR-005 (layout policy,
normative).
