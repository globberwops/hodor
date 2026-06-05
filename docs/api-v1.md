# Hodor v1 API Surface

This document captures the v1 public API as a single picture. The architectural decisions that produced this surface are recorded in `docs/adr/` and `CONTEXT.md`; this file is the *what*, not the *why*.

## Layout

- All public names are in the `hodor` namespace.
- All API entry points are **free functions** with the *context* as the first argument (`Map` for cold path, `QueryHandle` for hot path). No methods on `Map`, `QueryHandle`, or operation result types (`RoadState`, `LocateResult`, `Route`).
- **View types** (`RoadView`, `LaneView`, `SignalView`, `ObjectView`, `JunctionView`, `ControllerView`, `LaneSectionView`) are value types with methods. This is an explicit carve-out from ADR 0003.
- Hodor is **exception-free**. All fallible operations return `std::expected<T, E>`; all id-lookups return `std::optional<View>`.
- Hot-path functions are `noexcept`; fallible hot-path functions are `noexcept` and return `std::expected`.

## Build (cold, fallible)

```cpp
namespace hodor {

std::expected<Map, BuildError> Map::from_file  (const std::filesystem::path&);
std::expected<Map, BuildError> Map::from_buffer(std::span<const std::byte>);
const Diagnostics&              diagnostics    (const Map& m) noexcept;

}  // namespace hodor
```

## QueryHandle factory

```cpp
namespace hodor {
QueryHandle make_query_handle(const Map& m) noexcept;
// Consumer is responsible for `thread_local` storage if desired.
}  // namespace hodor
```

## Cold-path lookups (`std::optional<View>`)

```cpp
namespace hodor {

std::optional<RoadView>        road_of     (const Map& m, RoadId id)         noexcept;
std::optional<LaneView>        lane_of     (const Map& m, RoadId r, LaneId l) noexcept;
std::optional<SignalView>      signal_of   (const Map& m, SignalId id)       noexcept;
std::optional<ObjectView>      object_of   (const Map& m, ObjectId id)       noexcept;
std::optional<JunctionView>    junction_of (const Map& m, JunctionId id)     noexcept;
std::optional<ControllerView>  controller_of(const Map& m, ControllerId id)  noexcept;

}  // namespace hodor
```

## Cold-path iteration (`std::span<const View>`)

```cpp
namespace hodor {

std::span<const RoadView>      roads       (const Map& m) noexcept;
std::span<const JunctionView>  junctions   (const Map& m) noexcept;
// signals and objects are road-attached; iterate via `road_view.signals()` / `road_view.objects()`.

}  // namespace hodor
```

## Routing (cold, fallible, multi-waypoint)

```cpp
namespace hodor {

using DefaultWeightFn = double(*)(RoadId, RoadId);
constexpr DefaultWeightFn default_length_weight = /* … */;

struct Waypoint    { RoadId road; double s; };
struct RouteSegment{ RoadId road; double start_s; double end_s; bool forward; };
struct Route       { std::vector<RouteSegment> segments; double total_cost; };

enum class RouteError : std::uint8_t { Disconnected, SelfLoop, OutOfRange, InvalidWaypoint };

// Default weight (length)
std::expected<Route, RouteError> route(const QueryHandle& qh, std::span<const Waypoint> waypoints);

// Caller-supplied weight function (templated, inlinable)
template <typename WeightFn>
std::expected<Route, RouteError> route(const QueryHandle& qh, std::span<const Waypoint> waypoints,
                                       WeightFn weight_fn);

}  // namespace hodor
```

## Hot-path queries

```cpp
namespace hodor {

// ----- Position PODs (type-safe, aggregate-initializable) -----
struct InertialPos { double x; double y; };
struct RoadPos     { RoadId road; double s; double t = 0.0; };
struct LanePos     { RoadId road; LaneId lane; double s; };

// ----- Result types (POD-ish, no methods) -----
struct LocateResult { RoadId road_id; double s; double t; };
struct RoadState {
    double x, y, z;
    double h;             // heading (yaw) of the reference line at s
    double normal_x, normal_y;  // in-plane normal vector at s
    double slope;         // ds/dz equivalent
    double curvature;
    double banking;       // superelevation/crossfall effect
    double friction;
};

// ----- Locate (fallible: point may be off-network) -----
enum class LocateError : std::uint8_t { NoRoad, Ambiguous };

std::expected<LocateResult, LocateError> locate(const QueryHandle& qh, InertialPos p) noexcept;

// ----- Road surface state (primary hot path: inertial position in) -----
std::expected<RoadState, LocateError> road_state_at(const QueryHandle& qh, InertialPos p) noexcept;

// ----- Road surface state (OpenDRIVE-aware: road-frame position in) -----
RoadState road_state(const QueryHandle& qh, RoadPos pos) noexcept;

// ----- Lane width at s -----
double lane_width(const QueryHandle& qh, LanePos pos) noexcept;

// ----- Spatial queries (caller-bounded span) -----
std::size_t signals_in_radius(const QueryHandle& qh, InertialPos center, double r,
                              std::span<Signal> out) noexcept;
std::size_t objects_in_radius(const QueryHandle& qh, InertialPos center, double r,
                              std::span<Object> out) noexcept;

}  // namespace hodor
```

## View types (value types, with methods)

```cpp
namespace hodor {

class RoadView {
public:
    RoadId                                id()          const noexcept;
    double                                length()      const noexcept;
    std::optional<JunctionId>             junction()    const noexcept;
    std::optional<RoadId>                 predecessor() const noexcept;
    std::optional<RoadId>                 successor()   const noexcept;
    std::span<const GeometryPrimitive>    plan_view()   const noexcept;
    std::span<const LaneSectionView>      lane_sections() const noexcept;
    std::span<const SignalView>           signals()     const noexcept;
    std::span<const ObjectView>           objects()     const noexcept;
    std::optional<LaneView>               lane(LaneId)  const noexcept;
};

class LaneView {
public:
    LaneId       id()         const noexcept;
    LaneType     type()       const noexcept;
    bool         is_forward() const noexcept;     // driving direction
    // ...
};

class LaneSectionView { /* s_start, s_end, lanes (span) */ };
class SignalView     { /* id, type, position (road, s, t), orientation */ };
class ObjectView     { /* id, type, position, dimensions */ };
class JunctionView   { /* id, name, connecting_roads (span) */ };
class ControllerView { /* id, name, signals (span) */ };

}  // namespace hodor
```

## Error types

```cpp
namespace hodor {

// Cold-path, payload-bearing
struct BuildError {
    enum class Kind : std::uint8_t { MalformedXml, IoError, OutOfMemory };
    Kind              kind = {};
    std::uint32_t     xml_line   = 0;
    std::uint32_t     xml_column = 0;
    std::string_view  message;        // valid for the lifetime of the expected
};

// Hot-path, enum-only (no payload)
enum class LocateError  : std::uint8_t { NoRoad, Ambiguous };
enum class RouteError   : std::uint8_t { Disconnected, SelfLoop, OutOfRange, InvalidWaypoint };

}  // namespace hodor
```

## What's deliberately not in v1

- **No methods on `Map` or `QueryHandle`.** (ADR 0003.)
- **No dynamic-state layer.** (ADR 0001.)
- **No multi-file map merging.** (v1 scope; v2+ may add.)
- **No OpenDRIVE schema validation.** (ADR 0002 — builder is tolerant.)
- **No actual SIMD intrinsics in v1.** (v1 ships "SIMD friendly" data layout; v2 ships SIMD.)
- **No QH-side caches beyond `last_road_id_` in v1.** (v2 adds LRU + batch scratch.)
- **No batch queries in v1.** (v2 introduces them with SIMD.)
- **No lane-level routing in v1.** (ADR 0004 — road-level only.)
- **No bounding-box spatial queries.** (`signals_in_radius` is the only spatial query in v1.)
- **No `<junctionGroup>`, `<station>`, `<geoReference>`, `<userData>`, `<include>`, `<railroad>`, `<surface>` in v1 format coverage.**
- **No atomic-swap `Map` replacement in v1.** (The design enables it via `std::atomic<std::shared_ptr<T>>`; the API is not exposed.)

## Concurrency

Hodor's read path is **wait-free and lock-free**: every query operation completes in a bounded number of steps with no progress dependence on any other thread. Concurrent reads from any number of threads on any number of `Map` and `QueryHandle` instances are safe and never block. Construction and teardown are wait-free for the consumer (atomic refcount operations on `Map`); build itself is single-threaded.

**Read path guarantees:**
- Lock-free: no mutex, no rwlock, no spinlock in the hot path.
- Wait-free: every read is a bounded number of steps (load pointer → dereference → return). No CAS retry loop.
- Progress-independent: thread T reading cannot be delayed by thread U doing anything (build, query, swap, destroy).

**Construction guarantees:**
- Single-threaded in v1. Calling `Map::from_file` from two threads simultaneously is a user bug (UB).
- A v2 may add parallel parse; the "no concurrent build" rule is documented but not enforced in v1.

**`Map` lifetime guarantees:**
- Copy/clone/destroy are wait-free via `std::shared_ptr`'s atomic refcount.
- `Map` is a handle; the underlying `MapData` lives until the last `Map` (or `QueryHandle` holding a `Map` refcount) is dropped.

## Tooling (v1)

| Concern | Tool | Notes |
|---|---|---|
| Tests | `doctest` | Header-only, single-header, MIT, fast compile, modern C++-friendly. Tests under `tests/`. |
| Docs | Doxygen | `Doxyfile` at repo root, generates `docs/html/` and `docs/latex/`. |
| CI | GitHub Actions | Linux (GCC + Clang) and macOS (Apple Clang); ASan + UBSan on every push; TSan in nightly. |
| Static analysis | `clang-tidy` | Curated `.clang-tidy` (readability, modernize, performance, bugprone). |
| Sanitizers | ASan, UBSan, TSan | ASan+UBSan on every push; TSan in nightly. |
| Fuzzing | libFuzzer | The parser is the largest untrusted-input surface; corpus of real `.xodr` files. |
| Benchmarks | Google Benchmark | Micro-benchmarks for `locate`, `road_state_at`, `road_state`, `signals_in_radius`, `route` with regression tracking. |
| Code coverage | `lcov` / `genhtml` | CI generates coverage reports; tracked over time. |
| Formatting | `clang-format` | One `.clang-format` at repo root. CI fails the build on diff. |
| Git hooks | `pre-commit` (or `lefthook`) | Runs `clang-format` and a fast subset of `clang-tidy` on commit. |

**Library shape:**
- **Hot-path headers** (in `include/hodor/`) are header-only — templates, inlining, `noexcept`.
- **Cold-path internals** (Expat wrapper, `OpenDriveBuilder`, derived views) are compiled into a static lib.
- Consumers `find_package(Hodor)` and link against the imported target `Hodor::hodor`; hot-path headers are pulled in via `#include <hodor/hodor.hpp>`.

**Project layout (single CMake project):**
```
hodor/
├── CMakeLists.txt
├── Doxyfile
├── .clang-format
├── .clang-tidy
├── .pre-commit-config.yaml   (or lefthook.yml)
├── include/hodor/            (public API; hot-path headers are header-only)
│   └── hodor.hpp
├── src/                      (cold-path internals; compiled into Hodor::hodor)
│   ├── detail/expat_raii.hpp
│   ├── detail/builder.hpp
│   ├── detail/builder.cpp
│   └── …
├── tests/                    (doctest)
├── bench/                    (Google Benchmark)
├── fuzz/                     (libFuzzer harness for the parser)
├── examples/                 (load_and_locate, vehicle_dynamics_loop, route_planning, spatial_query, view_iteration)
├── docs/                     (this file, CONTEXT.md, adr/, agents/)
└── .github/workflows/        (CI)
```
