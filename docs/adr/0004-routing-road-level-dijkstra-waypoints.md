# Routing is road-level Dijkstra with multi-waypoint support and a custom weight hook

Hodor's routing is one of the named query APIs. We have decided:

1. **Granularity is road-level for v1.** Graph nodes are roads (`RoadId`); edges are transitions `(road_a, road_b)`. Lane-level routing is explicitly deferred to v2 (or later) — different lanes of the same road leading to different paths is real precision, but the graph size is 10–100x larger and Dijkstra cost scales accordingly.
2. **Algorithm is Dijkstra for v1.** O((V+E) log V) per query, no preprocessing, no parameters, always correct. OpenDRIVE networks are typically 10²–10⁴ roads; a single Dijkstra runs in microseconds. A* and Contraction Hierarchies are deferred to v2 — the *Contraction Hierarchies* path in particular is out of scope for v1 because of its preprocessing cost.
3. **The `RoutingGraph` is a derived view, eagerly built at `Map::build()` time.** Route queries are cold-path today but a future consumer may call them in inner loops; building the graph lazily on first route call would add an unpredictable first-call cost that violates the perf contract. The view is part of the immutable `Map` once built.
4. **Multi-waypoint support is in v1 from the start.** `route(qh, span<const Waypoint>)` runs Dijkstra between each consecutive pair of waypoints and concatenates the sub-routes. The total cost is the sum of sub-route costs. Failure on any sub-route fails the whole call with the offending pair's `RouteError`.
5. **Default weight is length.** Each edge's default weight is `road_b.length` (the cost of *entering* `road_b`). Consumers can override with a custom weight function for time-based routing, penalty-based avoidance, or any monotonic cost the consumer wants Dijkstra to optimize over.
6. **The route is materialized as a value the consumer owns.** No handles into the `RoutingGraph` view, no shared state. The `route()` call returns a fresh `std::vector<RouteSegment>` (inside a `Route` struct).

**API shape (locked):**

```cpp
namespace hodor {

struct Waypoint {
    RoadId road;
    double s;
};

struct RouteSegment {
    RoadId road;
    double start_s;     // where on `road` this segment starts
    double end_s;       // where on `road` this segment ends
    bool   forward;     // driving direction along the road
};

struct Route {
    std::vector<RouteSegment> segments;
    double total_cost;
};

enum class RouteError {
    Disconnected,    // no path between two consecutive waypoints
    SelfLoop,        // a waypoint is repeated with no intervening edge
    OutOfRange,      // invalid RoadId
    InvalidWaypoint, // s is outside [0, road.length]
};

using DefaultWeightFn = double(*)(RoadId, RoadId);  // function pointer, no allocation
constexpr DefaultWeightFn default_length_weight = +[](RoadId, RoadId to) {
    return road_length_of(to);   // resolved at build time
};

// Convenience overload: default weight (length)
std::expected<Route, RouteError>
route(const QueryHandle& qh, std::span<const Waypoint> waypoints);

// General form: caller-supplied weight function (templated, no type erasure)
template <typename WeightFn>
std::expected<Route, RouteError>
route(const QueryHandle& qh, std::span<const Waypoint> waypoints, WeightFn weight_fn);

}  // namespace hodor
```

**Why these specific decisions:**

- **Function pointer for the default weight (`DefaultWeightFn`):** zero-allocation, inlinable indirect call, no type erasure. The compiler can devirtualize in hot paths. The default just returns `road_b.length` resolved against the canonical `Map`.
- **Templated `WeightFn` for the general form:** preserves the perf contract — the consumer's lambda is inlined into Dijkstra's edge relaxation, no `std::function` overhead. The trade-off is that the function is a template, but since the entire Hodor API is header-only for the hot-path headers, this is consistent.
- **`std::span<const Waypoint>` for the waypoint list:** flexible (works with `std::vector`, `std::array`, stack arrays, initializer lists) and matches C++23 idiom. `std::initializer_list` would have been slightly more ergonomic at call sites but has lifetime gotchas we don't want to inherit.
- **Dijkstra with multi-waypoint = N×(Dijkstra between consecutive pairs):** simple, correct, and the per-pair Dijkstra dominates the cost. A multi-waypoint A* would need an admissible heuristic over waypoint chains, which is non-trivial; v1 stays with simple composition.
- **Direction-honoring by construction:** the routing graph's edges are directional — a one-way road contributes edges in its declared direction only. The route never goes "the wrong way" on a one-way because Dijkstra's edges don't allow it.

**The implications worth recording explicitly:**

- There is no "lane-level route" in v1, and there will not be one in v1. If a future contributor says "we should add lane-level routing," the answer is "open a v2 ticket; the graph and algorithm choices in this ADR are deliberate."
- The `RoutingGraph` view is built at `Map::build()` time using the canonical `Map`'s road links and junction connecting roads. It is a derived view, not stored in the canonical `Map` itself. This is consistent with the layered architecture (canonical + views).
- The `route()` function takes a `QueryHandle`, not a `Map`, so that future per-thread caching of recent routes (LRU of `(from, to) -> Route`) can be added to the `QueryHandle` without changing the API. v1 ships *no* such cache; the slot is reserved.
- One-way and junction topology are honored by the graph. **No other constraints** in v1: no vehicle-type, no speed-limit, no avoid-area, no time-of-day. Consumers compose those on top.
