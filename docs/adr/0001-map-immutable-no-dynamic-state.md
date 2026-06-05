# Map is immutable; Hodor has no dynamic-state layer

Hodor's `Map` is the in-memory model parsed from a single `.xodr` file. Consumers include vehicle dynamics, scenario engines, visualizers, and driver models. The perf contract requires thread-safe, lock-free reads after construction, with thread-local query handles for hot-path queries.

We have decided:

1. `Map` is **fully immutable** after `Map::build()` returns. There is no public mutation API on `Map` or any of its constituent types.
2. **Hodor does not provide a dynamic-state layer.** Placed actors, signal state, scenario events, lane overrides, and any other time-varying data are explicitly out of scope. Consumers that need them (notably scenario engines) own that state themselves and query Hodor's `Map` for the static geometry underneath.

The strict immutability gives us, essentially for free, the things the perf contract demands: lock-free reads, cheap cross-thread sharing, SoA-friendly immutable layouts, and no need for COW or atomic snapshots on the read path. A "live" model that scenario engines could mutate in place would have forced locking, COW, or atomic-swap machinery that contradicts the hot-path constraints. The cost is that dynamic state is *not* Hodor's job — any consumer that wants it must build (or use) their own layer and reference the `Map` by id.

The implication worth recording explicitly: scenario engines, vehicle dynamics, visualizers, and driver models **do not extend `Map`** and **must not call any mutation API on it**. If a future change wants dynamic state inside Hodor, the answer is "open a new project" — Hodor is the static map.
