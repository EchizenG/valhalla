# Valhalla Subset Rewrite Design

## 1) Overview of Valhalla

Valhalla is a routing stack built around a preprocessed road-graph tile dataset and a request pipeline that separates concerns across modules. In production usage, requests typically flow through:

- **Tyr** (API orchestration / request-response handling)
- **Loki** (input correlation: map user coordinates onto graph edges)
- **Thor** (graph search / pathfinding and derived algorithms)
- **Odin** (maneuver and narrative generation)
- **Tyr** (final serialization)

Key architectural properties relevant to a rewrite:

- **Tile-backed graph model** with hierarchy levels and graph-reader caching.
- **Dynamic costing** at runtime (costs are not precomputed into tiles).
- **Multiple endpoint types** (route, matrix, map matching, isochrone, locate, etc.) built from shared internals.
- **Separation of tile building vs. runtime service**, where Mjolnir prepares data and runtime modules consume it.

For a new rewrite project, the central decision is whether to preserve this modular decomposition or collapse some boundaries initially for delivery speed.

---

## 2) Major Valhalla modules

### Baldr
- **Role:** Core graph/tile data structures and graph reading.
- **Typical contents:** Graph identifiers, nodes, directed edges, edge metadata, tile access/cache.
- **Why it matters for rewrite:** This is the substrate used by almost every runtime function.
- **Rewrite priority:** **Very high** (foundational).

### Midgard
- **Role:** Geometry and spatial utility layer.
- **Typical contents:** Coordinate types, distance/shape math, projection helpers, encoded polyline handling.
- **Why it matters for rewrite:** Needed by correlation, path output geometry, and isochrone logic.
- **Rewrite priority:** **High**.

### Mjolnir
- **Role:** Data ingestion and tile construction from OSM and related datasets.
- **Typical contents:** OSM parsing pipeline, graph building/enhancement/filtering, hierarchy/shortcut construction.
- **Why it matters for rewrite:** Required only if you must produce your own tileset.
- **Rewrite priority:** **Medium to high** depending on data strategy.

### Loki
- **Role:** Location correlation and input normalization.
- **Typical contents:** Candidate edge search near coordinates, filtering and reachability checks.
- **Why it matters for rewrite:** Nearly all coordinate-based APIs depend on robust correlation.
- **Rewrite priority:** **High**.

### Sif
- **Role:** Runtime costing and access logic.
- **Typical contents:** Mode-specific cost models and edge/transition permission checks.
- **Why it matters for rewrite:** Determines route behavior quality and mode semantics.
- **Rewrite priority:** **High**.

### Thor
- **Role:** Path computation and other heavy graph algorithms.
- **Typical contents:** Shortest path variants, matrix computation, isochrone expansion, trace route support.
- **Why it matters for rewrite:** Main algorithmic engine.
- **Rewrite priority:** **Very high**.

### Odin
- **Role:** Guidance/maneuver narration generation.
- **Typical contents:** Turn instruction logic and localization-oriented narrative formatting.
- **Why it matters for rewrite:** Critical for turn-by-turn UX, not required for raw-path engines.
- **Rewrite priority:** **Medium**.

### Tyr
- **Role:** API orchestration and serialization.
- **Typical contents:** Endpoint handlers, request dispatching, output formats.
- **Why it matters for rewrite:** Needed for service exposure, optional for library-only prototype.
- **Rewrite priority:** **Medium** initially.

### Meili
- **Role:** Map matching.
- **Typical contents:** Candidate search integration + probabilistic sequence matching.
- **Why it matters for rewrite:** Only necessary if trace/map-matching products are in scope.
- **Rewrite priority:** **Medium/late** unless map matching is a primary goal.

### Skadi
- **Role:** Elevation/DEM integration.
- **Typical contents:** Height lookups and elevation profile support.
- **Why it matters for rewrite:** Required for elevation endpoint and elevation-aware outputs.
- **Rewrite priority:** **Low to medium** for most minimal route-focused rewrites.

---

## 3) Public service/API functions

> Note: naming in docs and deployed APIs sometimes differs (`sources_to_targets` vs “matrix”, `height` vs “elevation”, etc.). Endpoint and payload compatibility details should be validated during implementation planning (**needs source inspection**).

### route
- **Purpose:** Compute turn-by-turn route between ordered locations.
- **Input:** Ordered coordinate locations, costing mode/options, optional time-dependent settings.
- **Output:** Path geometry + legs + summary, optionally maneuvers/instructions.
- **Main modules:** Tyr, Loki, Thor, Sif, Baldr, Midgard, Odin.
- **Core algorithm/data structure:** A*/bidirectional A* variants over tiled directed graph with runtime cost model.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** No hard API dependency, but shares internals with matrix/isochrone/trace.

### matrix
- **Purpose:** Compute travel time/distance for source-target combinations.
- **Input:** Source and target coordinate sets, costing mode/options.
- **Output:** Matrix of costs/times/distances; may include null/unreachable entries.
- **Main modules:** Tyr, Loki, Thor, Sif, Baldr.
- **Core algorithm/data structure:** Many-to-many graph expansion with reusable frontier/state.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** Often reuses Thor search primitives used by route.

### optimized_route
- **Purpose:** Compute route with stop-order optimization (TSP-like behavior).
- **Input:** Multiple locations plus optimization and costing options.
- **Output:** Reordered stops and resulting route details.
- **Main modules:** Tyr, Loki, Thor, Sif, Baldr, Odin.
- **Core algorithm/data structure:** Stop permutation/optimization + repeated shortest-path calls.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** Strongly depends on route/matrix primitives.

### isochrone
- **Purpose:** Generate reachable region(s) from origin under time/distance budgets.
- **Input:** Origin(s), contours (time or distance), costing options.
- **Output:** Polygon/contour geometry representing reachable area.
- **Main modules:** Tyr, Loki, Thor, Sif, Baldr, Midgard.
- **Core algorithm/data structure:** Cost-limited graph expansion then contour/polygon construction.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** Shares expansion and costing internals with route/matrix.

### trace_route
- **Purpose:** Map-match a trace and return route-like trip output.
- **Input:** Ordered GPS trace points (optionally timestamps/accuracy), costing options.
- **Output:** Matched route geometry and directions-like output.
- **Main modules:** Tyr, Loki, Meili, Thor, Sif, Baldr, Odin.
- **Core algorithm/data structure:** Candidate generation + sequence inference (HMM/Viterbi-style) + route construction.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** Depends on map matching internals and route/path construction.

### trace_attributes
- **Purpose:** Map-match a trace and return edge-level metadata rather than narrative route.
- **Input:** GPS trace points and matching options.
- **Output:** Matched edge attributes and diagnostic metadata.
- **Main modules:** Tyr, Loki, Meili, Thor, Sif, Baldr.
- **Core algorithm/data structure:** Same matching core as trace_route, different serializer/output model.
- **Rewrite difficulty:** **High**.
- **Depends on other functions:** Depends on map matching core; may share output shaping with route internals.

### locate
- **Purpose:** Return graph information near input location(s).
- **Input:** Coordinate location(s), optional search/radius filters.
- **Output:** Nearby edge/node metadata and correlation details.
- **Main modules:** Tyr, Loki, Baldr, Midgard.
- **Core algorithm/data structure:** Spatial bin/candidate lookup over tiled graph.
- **Rewrite difficulty:** **Medium**.
- **Depends on other functions:** No strict dependency, but reused by route/trace preprocessing patterns.

### elevation
- **Purpose:** Return elevation for points or along a shape.
- **Input:** Points or polyline/shape definition.
- **Output:** Heights/elevation profile.
- **Main modules:** Tyr, Skadi, Midgard.
- **Core algorithm/data structure:** DEM raster lookup and sampling/interpolation.
- **Rewrite difficulty:** **Medium**.
- **Depends on other functions:** Mostly independent from route core.

### expansion
- **Purpose:** Debug endpoint exposing explored graph during routing search.
- **Input:** Route-like request plus expansion settings.
- **Output:** Explored edge/node set or expansion traces.
- **Main modules:** Tyr, Loki, Thor, Sif, Baldr.
- **Core algorithm/data structure:** Instrumented graph search traversal output.
- **Rewrite difficulty:** **Medium**.
- **Depends on other functions:** Depends on route/path search internals.

### status
- **Purpose:** Health and dataset/service metadata.
- **Input:** Usually minimal/no route input.
- **Output:** Service version, loaded tileset metadata, capabilities.
- **Main modules:** Tyr (and config/runtime state from others).
- **Core algorithm/data structure:** Service metadata aggregation.
- **Rewrite difficulty:** **Low**.
- **Depends on other functions:** No.

### centroid
- **Purpose:** Compute centroid-related output (API exposure/behavior variant requires validation).
- **Input:** **Needs source inspection** (not uniformly documented with other core endpoints).
- **Output:** **Needs source inspection**.
- **Main modules:** Likely Tyr + Midgard (and possibly Thor/Loki depending on semantics).
- **Core algorithm/data structure:** Geometric centroid calculation or region-derived centroid (**needs source inspection**).
- **Rewrite difficulty:** **Low to medium** (tentative).
- **Depends on other functions:** Likely independent, but **needs source inspection**.

---

## 4) Internal architecture diagram (text)

```text
[Client/Caller]
      |
      v
    [Tyr API Layer]
      |
      +--> [Loki Correlation] ----> [Baldr GraphReader + Tiles]
      |           |                          ^
      |           v                          |
      |        [Midgard Geometry]            |
      |
      +--> [Thor Path/Matrix/Isochrone] <----+
      |           |
      |           v
      |        [Sif Costing]
      |
      +--> [Meili Matching] (trace endpoints)
      |
      +--> [Odin Narration] (route-like guidance output)
      |
      +--> [Skadi Elevation] (elevation endpoint / optional profile)
      |
      v
[Serialized Response (JSON/pbf/etc.)]

Offline pipeline (separate runtime concern):
[OSM + aux data] --> [Mjolnir tile build pipeline] --> [Baldr-consumable tile dataset]
```

---

## 5) Dependency map between modules

```text
Baldr   <- foundational graph/tile layer used by Loki, Thor, Sif, Meili, Tyr
Midgard <- geometry utilities used by Loki/Thor/Odin/Skadi/Tyr
Sif     <- costing policy used primarily by Thor (and matching/path decisions)
Loki    <- depends on Baldr + Midgard; upstream for most route-like APIs
Thor    <- depends on Baldr + Sif + Midgard; optional inputs from Loki/Meili
Meili   <- depends on Loki-like candidate search + Thor/Baldr/Sif primitives
Odin    <- depends on Thor output + Midgard + naming/metadata from Baldr
Skadi   <- mostly independent elevation subsystem; used by Tyr and optionally routes
Tyr     <- orchestrates all runtime modules and owns endpoint surface
Mjolnir <- offline producer of tiles consumed by Baldr at runtime
```

---

## 6) Minimal rewrite options

### A) Shortest-path-only engine
**Scope:** basic route path/cost between two points, limited modes, no maneuvers.

- Must rewrite first: Baldr-like graph core, Midgard-lite geometry, Loki-lite correlation, Sif-lite costing, Thor-lite search.
- Can defer: Odin, Meili, Skadi, advanced Tyr features, Mjolnir (if reusing existing tiles).
- Value: fastest path to a usable routing core.

### B) Route + matrix engine
**Scope:** production-relevant route and many-to-many cost matrix.

- Adds on top of A: robust matrix algorithm in Thor, stronger request validation in Tyr.
- Can still defer: full narration depth, map matching, elevation.
- Value: supports many logistics and planning workloads.

### C) Map-matching engine
**Scope:** trace_route and trace_attributes quality comparable baseline.

- Requires: strong Loki candidate search + Meili-like probabilistic matcher + route reconstruction.
- Can defer: full optimized_route and some narrative sophistication.
- Value: telemetry and GPS cleanup products.

### D) Full routing engine
**Scope:** near-complete parity including optimized route, isochrone, trace, elevation, expansion, status.

- Requires all major modules, plus significant test surface and data-pipeline maturity.
- Value: maximum feature completeness, slowest/riskiest path.

---

## 7) Recommended rewrite order

1. **Data contract decision**
   - Reuse Valhalla tiles initially or define new tile/storage format.
   - This decision gates Baldr/Mjolnir scope.

2. **Baldr-core + Midgard-core**
   - Graph primitives, tile reading, geometry essentials.

3. **Loki-core**
   - Reliable coordinate→edge correlation with deterministic behavior.

4. **Sif-core**
   - Minimal but extensible dynamic costing for target modes.

5. **Thor route core**
   - Shortest path algorithm and path reconstruction.

6. **Tyr minimal API**
   - Expose route + status first; keep interface thin.

7. **Thor matrix + Tyr matrix endpoint**
   - Reuse route primitives; optimize frontier reuse.

8. **Optional branch paths**
   - **If map matching needed:** Meili + trace endpoints.
   - **If guidance needed:** Odin narratives.
   - **If geospatial analytics needed:** isochrone and expansion.
   - **If elevation needed:** Skadi endpoint integration.

9. **Only after runtime stability:**
   - Mjolnir rewrite for custom data production.

---

## 8) Components that can be skipped initially

- Odin full maneuver localization stack.
- Skadi/elevation endpoint.
- optimized_route stop-order optimization.
- expansion debug endpoint.
- Full output-format parity (e.g., all serializers/protobuf variants).
- Mjolnir full planet-scale build pipeline (if bootstrapping from existing prepared data).

---

## 9) Components that are hard to rewrite

- **Mjolnir full tile build parity:** OSM normalization, relations/restrictions, hierarchy, shortcuts at scale.
- **High-quality correlation (Loki):** edge candidate ranking/reachability corner cases.
- **Mode-accurate costing (Sif):** subtle policy interactions and performance constraints.
- **Map matching (Meili):** probabilistic inference with robust fallback behavior.
- **Isochrone quality/performance:** contour correctness and runtime scaling.
- **Narrative parity (Odin):** instruction quality and multilingual output.

---

## 10) Risks and unknowns

1. **Tile/data compatibility risk**
   - Reusing existing tiles accelerates delivery but constrains internal design.
   - New format gives freedom but increases scope drastically.

2. **Algorithm parity risk**
   - Matching route quality and edge-case behavior is harder than basic shortest-path correctness.

3. **Performance risk**
   - Memory/cache behavior and graph traversal efficiency dominate runtime at scale.

4. **Scope creep risk**
   - Adding parity features too early (optimized route, full narration, all endpoints) can stall core delivery.

5. **Documentation-to-code gap**
   - Some endpoint specifics and option semantics require direct source-level confirmation.
   - Marked as **needs source inspection** where unclear.

6. **Testing/data risk**
   - Without a strong golden test corpus (API snapshots + map fixtures), regressions will be hard to detect.

---

## 11) Suggested phase-by-phase implementation plan

### Phase 0 — Decision & contracts
- Freeze target endpoint subset and non-goals.
- Decide data strategy (reuse tiles vs new format).
- Define internal interfaces for graph reader, costing, and search.

### Phase 1 — Core runtime substrate
- Implement Baldr-like graph primitives and reader abstraction.
- Implement Midgard-like geometry primitives required by route output.
- Build deterministic unit tests for graph traversal primitives.

### Phase 2 — Correlation + basic routing
- Implement Loki-like nearest-edge/candidate pipeline.
- Implement Sif-lite costing for one mode (e.g., auto).
- Implement Thor route search and path assembly.
- Expose `route` + `status` via minimal Tyr-like service.

### Phase 3 — Matrix capability
- Add Thor many-to-many computation and endpoint wiring.
- Add performance profiling and cache strategy refinement.
- Harden failure semantics and unreachable-cell behavior.

### Phase 4 — Product branch (choose one)
- **Map matching track:** add Meili-style matching + `trace_route`/`trace_attributes`.
- **Directions track:** add Odin maneuver generation and narration quality.
- **Analytics track:** add `isochrone` and optional `expansion`.

### Phase 5 — Secondary endpoints and enhancements
- Add `optimized_route`, `locate`, `elevation`, and output-format extensions as needed.
- Expand costing modes and option compatibility.

### Phase 6 — Data pipeline (optional but often eventual)
- Rebuild Mjolnir-equivalent ingest pipeline only if independent data production is required.
- Validate large-region/planet-scale build/runtime economics.

---

## 12) Practical rewrite guidance (what to do first vs later)

### Rewrite first (high leverage)
1. Graph data access abstraction (Baldr-like).
2. Coordinate correlation (Loki-like) with measurable confidence/ranking.
3. Path search + costing loop (Thor + Sif core).
4. Minimal API shell for `route` and `matrix`.

### Rewrite later (defer safely)
1. Full narrative/maneuver sophistication (Odin).
2. Elevation and niche endpoints.
3. Stop-order optimization and advanced analytics.
4. Full custom tile build pipeline.

### Always keep explicit in planning
- Any behavior not validated against source/docs should remain tagged **needs source inspection** until confirmed.
- Avoid coupling early architecture to parity assumptions that are not yet tested.
