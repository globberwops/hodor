# Functional-style API: free functions in the `hodor` namespace

The Hodor public API is exposed as **free functions in the `hodor` namespace**. The first argument of every function is the *context*: a `const Map&` for cold-path queries, a `const QueryHandle&` for hot-path queries. There are no methods on `Map`, `QueryHandle`, or any of the query-result types. Fallible operations return `std::expected<T, E>`; non-fallible operations return `T` directly.

Concrete shape:

```cpp
namespace hodor {

    // Cold path — Map is the context
    const Road&         road_of      (const Map& m, RoadId id);
    LaneAttributes      attributes_of(const Map& m, RoadId id, LaneSectionId ls, LaneId lane);

    // Hot path — QueryHandle is the context
    RoadState           road_state   (const QueryHandle& qh, RoadId id, double s, double t = 0.0);
    std::expected<LocateResult, LocateError>
                        locate       (const QueryHandle& qh, double x, double y);
}
```

**Why this shape:**

1. **Explicit hot/cold split.** The call site names which layer it is hitting. A reviewer greping for `hodor::road_state(` can immediately see whether it is a cold lookup or a hot per-step query — that distinction is the heart of the perf contract, and making it visible at the call site is the cheapest way to keep it honest.
2. **No methods means no surprise state.** A `QueryHandle` looks like a "context" passed in, not an object with hidden mutable behavior. This makes the lock-free guarantee easier to prove and audit: a function with no method receiver cannot mutate receiver state in surprising ways.
3. **Composability.** Free functions compose with `std::bind`, `std::function`, ranges, and lambdas without `friend` declarations or method-pointer awkwardness.
4. **ADL-friendly.** Consumers can extend with their own free functions in their own namespaces, finding `hodor` types via ADL.
5. **Symmetric with the layers.** Free functions map 1:1 onto the three layers (`Map` for canonical+views, `QueryHandle` for per-thread). Methods would tempt us to put hot/cold on the same object.

**Argument order is fixed: context first, then ids, then numerics.** `road_state(qh, road_id, s, t)` is the only acceptable ordering; `road_state(road_id, s, t, qh)` is rejected. Context-first means the call site is readable left-to-right: "with this context, on this id, at this position."

**The `Map` is a handle type with internal `shared_ptr<const MapData>`.** Copying a `Map` is a refcount bump; the underlying data is shared and never mutated. This shape **intentionally** enables lock-free atomic replacement of the `Map` in a future version via `std::atomic<std::shared_ptr<T>>` (C++20) — useful for map streaming, tile swapping, or hot-reload of `.xodr` files. v1 does not expose any mutation or swap; `Map` is single, immutable, no-reload. But the design leaves the door open without API change.

**`QueryHandle` is created by the free factory function `hodor::make_query_handle(const Map&)`.** The consumer is responsible for `thread_local`-ifying it (or whatever lifetime strategy they want). Hodor does not bake in a thread model. The expected v1 use is `thread_local auto qh = hodor::make_query_handle(map);` at the top of a worker function.

**Fallible operations return `std::expected<T, E>`.** The `E` is a small error enum scoped to the operation (`LocateError::NoRoad`, `LocateError::Ambiguous`, `BuildError::MalformedXml`, …). Non-fallible operations return `T` directly. We rely on the type system: if the caller already has a `RoadId`, `road_state(qh, id, s, t)` is non-fallible (the id is valid by construction); if the caller has only `(x, y)`, `locate(qh, x, y)` is fallible.

**The implication worth recording explicitly:** there will be no `Map::road()` method, no `QueryHandle::road_state()` method, and no `Road::attributes()` method. The free-function form is the *only* public API. Anyone proposing a method on these types is wrong, and the answer is "this is recorded in ADR 0003, please add a free function."

**Carve-out for `View` types:** the "no methods" rule above applies to *context* types (`Map`, `QueryHandle`) and *operation result* types (`RoadState`, `LocateResult`, `Route`, `Waypoint`, `RouteSegment`). It does **not** apply to `View` types (`RoadView`, `LaneView`, `SignalView`, `ObjectView`, `JunctionView`, `ControllerView`, `LaneSectionView`). View types are value types representing canonical data — closer in spirit to `std::span<T>` than to a context. They have methods (`road_view.id()`, `road_view.length()`, `road_view.lane(id)`, `road_view.signals()`, …) following the `std::span<T>` convention. This carve-out is deliberate: the no-methods rule is about preventing the *context* types from gaining hidden mutable behavior; View types are pure data accessors and methods on them are the natural shape.
