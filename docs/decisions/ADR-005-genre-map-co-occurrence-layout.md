# ADR-005: Genre map layout from the user's own co-occurrence graph

- Status: Accepted (2026-07-10)
- Deciders: Fable (design)

## Context

The "genre map" must place genres in a 2D space where proximity feels meaningful. Published
genre atlases (musicmap.info, Every Noise at Once) are proprietary or Spotify-derived; bundling
their coordinates poses licensing risk and permanent staleness. MusicBrainz provides genre
*labels* per artist but no spatial layout.

## Decision

Build the map **from the user's own library**:

1. Nodes = genres having listened weight > 0 in the selected date range (top 150 by weight;
   remainder aggregated into an "other" list shown in a side panel, never silently dropped).
2. Edge weight between genres A and B = the summed listening metric of artists tagged with both
   A and B (co-occurrence through shared artists).
3. Layout = d3-force simulation (link force by edge weight, charge repulsion, collision by node
   radius) run client-side with a **seeded PRNG and fixed iteration count**, so the same data
   always yields the same map; positions cached keyed by a hash of the input.
4. Node size ∝ listening metric (plays or minutes); node color from a fixed categorical palette
   hashed by genre name (stable across sessions).

## Consequences

- Zero licensing exposure; the map is personal by construction ("your musical territory"), which
  matches the product concept better than a universal atlas.
- Sparse libraries yield sparse maps — acceptable; the empty/thin states are specified.
- Layout quality is a known unknown (ISSUE_PLAN "Known unknowns"): if force layout produces
  hairballs at 150 nodes, fallback knobs are documented (edge-weight pruning to a maximum
  spanning tree + k strongest extra edges).
- Deterministic rendering makes E2E screenshot assertions feasible.

## Alternatives considered

- **Bundled static atlas coordinates**: licensing risk, stale, impersonal; rejected.
- **Server-side layout**: portable results, but duplicates d3-force server-side for no user
  benefit at this node count; rejected.
- **UMAP/t-SNE over genre embedding**: needs an embedding source (none keyless) and adds a
  heavyweight dependency; rejected for v1.
