# Hodor

Hodor is a library for parsing ASAM OpenDRIVE 1.9 (`.xodr`) files into an in-memory model and providing query APIs for downstream consumers (vehicle dynamics, scenario engines, visualizers, driver models). The hot path is optimized for SoA layouts, SIMD-friendly evaluation, and lock-free reads after construction.

## Language

**Map**:
The in-memory model of a single `.xodr` file, built once and immutable for the rest of its lifetime. `Map` is a handle type with internal `shared_ptr<const MapData>` — copying a `Map` is a refcount bump, the underlying data is shared across threads and never mutated. The shared-ptr shape intentionally enables lock-free atomic replacement via `std::atomic<std::shared_ptr<T>>` in a future version (map streaming / tile swap / hot-reload); v1 does not expose any swap or reload.
_Avoid_: world, scene, network, level

**QueryHandle**:
A per-thread context for hot-path queries, built from a `Map` via `hodor::make_query_handle(const Map&)`. Holds caches and scratch buffers local to one thread; not shareable across threads. Consumers typically store a `QueryHandle` in `thread_local` storage. See ADR 0003.
_Avoid_: session, context, query

**MapData**:
The shared, immutable heap-allocated state behind a `Map`. The atomic-refcounted `shared_ptr<const MapData>` inside `Map` is the only shared mutable state in Hodor (the refcount itself). The pointed-to `MapData` is never mutated after `Map::build()` returns. All derived views (routing graph, spatial index) live inside `MapData`.
_Avoid_: scene_data, map_impl, internals

**RoadLocator, RoutingGraph, SignalObjectIndex** (derived views):
Three precomputed structures built inside `MapData` at `Map::build()` time. They are not separate types exposed to consumers; they are internal fields of `MapData` accessed only through the query functions (`locate`, `route`, `signals_in_radius`, `objects_in_radius`). Each is built once, immutable after. The "derived view" idea is an internal implementation pattern, not a public API.
_Avoid_: index, graph, table

**View** (`RoadView`, `LaneView`, `SignalView`, `ObjectView`, `JunctionView`, `ControllerView`, `LaneSectionView`):
Value types representing a non-owning, read-only view into the canonical `Map`'s data. Returned from id-lookups (`road_of`, `lane_of`, etc.). Cheap to copy (a few pointers). Have methods (e.g., `road_view.id()`, `road_view.lane_sections()`), following the `std::span<T>` convention. This is an explicit carve-out from ADR 0003's "no methods" rule, which applies to context types (`Map`, `QueryHandle`) and operation result types (`RoadState`, `LocateResult`, `Route`), not to value-type views.
_Avoid_: handle, ref, proxy

**InertialPos**:
POD with `double x, y`. A position in the `.xodr` file's INERTIAL frame. Used as input to inertial-space hot-path queries (`locate`, `road_state_at`, `signals_in_radius`, `objects_in_radius`). Aggregate-initializable, no methods.
_Avoid_: world_pos, vec2

**RoadPos**:
POD with `RoadId road; double s; double t = 0.0`. A position on a road in (s, t) coordinates. `t = 0` is the reference line (OpenDRIVE "center of road" convention). Used as input to road-frame hot-path queries (`road_state`).
_Avoid_: lane_pos, road_point

**LanePos**:
POD with `RoadId road; LaneId lane; double s`. A longitudinal position within a specific lane. Used as input to `lane_width`. The lateral `t` is irrelevant; lane identity is `LaneId`, not `t`.
_Avoid_: lane_coord, lane_ref

**Waypoint**:
POD with `RoadId road; double s`. A waypoint on a road, used as input to multi-waypoint `route` queries. Distinct from `RoadPos` because routing has no meaningful `t`; routes follow road centerlines and adjacent-road decisions are made at junctions.
_Avoid_: via_point, stop

**RoadState, LocateResult, Route, RouteSegment, BuildError, LocateError, RouteError**:
Result and error types of fallible / hot-path operations. POD-ish, no methods (per ADR 0003). Detailed shapes in `docs/api-v1.md`.

**v1 QueryHandle contents (note, not a term)**:
In v1 the `QueryHandle` is intentionally minimal: a `last_road_id_` fast path (consecutive same-road calls skip the road lookup and spatial-index probe), a stable pointer to the cached road geometry, and a `Map` refcount hold. No LRU caches, no batch scratch buffer, no routing context — those are deferred to v2. Total size under 64 bytes.

## Scope

Static map only. Dynamic state (placed actors, signal state, scenario events, lane overrides) is **explicitly out of scope**; consumers that need it own that state themselves. See ADR 0001.

In v1, a `Map` is built from exactly one `.xodr` file. Multi-file merging is out of scope for v1; this is a v1 simplification, not a permanent rule.

OpenDRIVE schema validation is out of scope; the builder is tolerant. See ADR 0002.

**v1 format coverage**: the canonical `Map` parses and stores `<road>` (with `<planView>`, `<lanes>` incl. lane sections / lane widths / road marks, `<elevationProfile>`, `<lateralProfile>`, `<objects>`, `<signals>`), `<controller>`, and `<junction>`. Deferred to v2: `<junctionGroup>`, `<station>`, `<geoReference>`, `<userData>`, `<include>`, `<railroad>`, `<surface>`. Deferred elements are silently consumed by the SAX handler (start/end events balanced, nothing stored). Unknown elements are also silently consumed.

**v1 diagnostics**: a `Diagnostics` struct is attached to every `Map` and is populated during the build pass. It exposes counters of skipped elements by category and a list of `{line, column, element_name}` entries for unknown elements seen. Accessed via `hodor::diagnostics(const Map&)`; immutable after build. This is in v1 scope because real-world `.xodr` files regularly contain vendor extensions and the debug signal is high-value from day one.

**v1 coordinate-frame principle**: Hodor works entirely in the `.xodr` file's INERTIAL frame. There is no built-in ENU/global frame concept; no origin offset, no rotation, no second frame. Consumers that have a different frame do their own conversions. The "coordinate transformations" the API exposes are within the INERTIAL frame: `(s, t) ↔ (x, y, z, h, normal)`, plus the inverse `(x, y) → (road_id, s, t)`.

**v1 routing is road-level Dijkstra with multi-waypoint support and a custom weight hook.** Graph nodes are roads; the `RoutingGraph` is a derived view built eagerly at `Map::build()` time. The default weight is `road_b.length`; consumers can supply a custom weight fn for time-based, penalty-based, or any monotonic cost. Multi-waypoint routes are composed of Dijkstra runs between consecutive waypoints. See ADR 0004.

**Latency tiers (note, not a term)**:
The API surface is binary (`Map` for cold path, `QueryHandle` for hot path), but consumer workloads sit on a latency spectrum:
- **Tight tier** (vehicle dynamics): ~100–1000 calls per 1 ms; tightest per-call budget; SIMD- and cache-sensitive.
- **Warm tier** (driver model perception): a few hundred calls per 4 ms; per-call budget more relaxed; locality patterns differ from tight tier.
- **Cold tier** (scenario engine, visualizer, route planning): event-driven; per-call latency not budgeted.

In v1, **tight and warm tier both go through `QueryHandle`** with the same minimal cache. Tier-aware QH caching (e.g., LRU of `(x, y) → road_id` for warm tier, last-probed-cell for spatial queries) is a v2 perf concern, not a v1 API change.

**v1/v2 perf split (note, not a term)**:
- **v1** ships AoS everywhere, "SIMD friendly" data layout (contiguous, aligned, branchless hot loops), and **no actual SIMD intrinsics**. A `HODOR_ENABLE_SIMD` CMake option is reserved but off in v1.
- **v2** introduces SoA for batch-query data (lane attributes, etc.) **and** actual SIMD intrinsics **in the same release** that adds the QH's `batch_scratch_`. The QH gains per-thread caches for warm-tier workloads (LRU of `(x, y) → road_id`, last-probed-cell) at the same time.

**Error reporting (note, not a term)**:
Hodor is **exception-free**. Errors are values:
- **Fallible operations** return `std::expected<T, E>`, with per-op `E` types (`BuildError`, `LocateError`, `RouteError`). Hot-path `E` is `enum class : std::uint8_t`; cold-path `E` is a struct with `Kind` + payload (e.g., `BuildError` carries Expat's `xml_line` / `xml_column` / `message`).
- **Lookups** that "give me an entity by id" return `std::optional<View>` (e.g., `road_of`, `lane_of`, `signal_of`, `object_of`, `junction_of`, `controller_of`). `std::nullopt` means not found. The `View` types are first-class value types, not `reference_wrapper` or raw pointers.
- **Hot-path functions are `noexcept`**. They trust the caller. See ADR 0005.

The API is **functional-style**: free functions in the `hodor` namespace, first argument is the context (`Map` for cold, `QueryHandle` for hot). No methods on `Map`, `QueryHandle`, or query-result types. See ADR 0003.

## Technology

C++23, modern CMake (≥ 3.30 baseline, updated as needed). Standard library used heavily (`std::expected`, `std::mdspan`, `std::flat_map`, `std::span`, `std::unreachable`). Compiler baseline: latest stable GCC and Clang.

Dependencies: minimal, used only where the effort of hand-rolling would exceed the cost of integration. The XML parser is **Expat** (C, MIT, used via a small internal RAII wrapper — no third-party C++ wrapper). Everything else (SIMD wrapper, spatial index, routing graph) is hand-rolled. See ADR 0002.

**Tooling (v1)**:
- Tests: **doctest** (header-only, MIT, modern).
- Docs: **Doxygen** with one `Doxyfile` at the repo root.
- CI: **GitHub Actions** on Linux (GCC + Clang) and macOS (Apple Clang); ASan + UBSan on every push; TSan in nightly.
- Static analysis: **clang-tidy** with a curated `.clang-tidy`.
- Sanitizers: **ASan + UBSan** in CI; **TSan** nightly.
- Fuzzing: **libFuzzer** for the parser with a real-file corpus.
- Benchmarks: **Google Benchmark** for hot-path micro-benchmarks with regression tracking.
- Code coverage: **lcov** + **genhtml** in CI.
- Formatting: **clang-format** (one `.clang-format` at the repo root, CI fails on diff).
- Git hooks: **pre-commit** (or lefthook) running clang-format and a fast subset of clang-tidy on commit.

**Library shape**:
- Hot-path headers (`include/hodor/`) are **header-only** — templates, inlining, `noexcept`.
- Cold-path internals (Expat wrapper, `OpenDriveBuilder`, derived views) are compiled into a **static lib** `Hodor::hodor`.
- Consumers `find_package(Hodor)` and link `Hodor::hodor`; include `<hodor/hodor.hpp>` for the hot-path API.
