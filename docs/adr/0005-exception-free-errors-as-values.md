# Hodor is exception-free; all errors are communicated as `std::expected` or `std::optional`

Hodor communicates errors through the type system, never through exceptions. This is a load-bearing part of the perf contract: exceptions introduce codegen for stack unwinding, increase binary size, and add branches the perf-sensitive hot path cannot afford.

We have decided:

1. **No exceptions, ever.** Hodor does not `throw`, and consumers cannot `catch` anything Hodor raises. The build is compiled with `-fno-exceptions` (or the MSVC equivalent). There is no exception-aware code path in the hot path. The library is exception-free by source: no `throw` keyword anywhere in `hodor/`, and no C++ standard library component that may throw is used on the hot path.
2. **Fallible *operations* return `std::expected<T, E>`.** `E` is a per-op type. Hot-path ops (`locate`, `route`) carry a small `enum class : std::uint8_t`. Cold-path ops (`Map::from_file`, `Map::from_buffer`) carry a struct with a `Kind` enum + payload (line/column/message from Expat). The success value of the hot-path `expected` is a small struct (`LocateResult`, `Route`); the failure is a single byte. This keeps the success path small, inlinable, and register-resident.
3. **Lookups that "give me an entity by id" return `std::optional<View>`.** The success value is a small, non-owning view type (`RoadView`, `LaneView`, `SignalView`, `ObjectView`, `JunctionView`, `ControllerView`). The view is cheap to copy (a few pointers) and gives access to the canonical entity's data through methods and `std::span` accessors. `std::nullopt` means "no such id." The view is *not* a `std::reference_wrapper` — it is a first-class value type, which lets us evolve the view (add methods, cache fields) without changing the lookup signature.
4. **Per-op `E` types, not a unified hierarchy.** `BuildError`, `LocateError`, `RouteError` are distinct types. No `std::error_code`, no `std::system_error`, no virtual dispatch. `std::expected` is monomorphic in `E`; the caller pattern-matches on the enum with a `switch`.
5. **No aborts, no terminates.** Hodor does not call `std::abort`, `std::terminate`, or `std::unexpected`. Every error is recoverable; the caller decides what to do. Internal invariants are checked with `std::unreachable` (C++23) for "this cannot happen" branches in the hot path — `std::unreachable` is undefined behavior, not a panic, and we use it only when the failure mode is a programming error in Hodor itself.
6. **Hot-path functions are `noexcept`.** They trust the caller (e.g., `signals_in_radius` is `noexcept`; passing an invalid span is a programming error). Cold-path functions are not `noexcept` and return `std::unexpected` on failure.

**Concrete shape (this is what the rule looks like at the call site):**

```cpp
namespace hodor {

// Cold-path lookups: std::optional<View>
std::optional<RoadView>        road_of     (const Map& m, RoadId id)         noexcept;
std::optional<LaneView>        lane_of     (const Map& m, RoadId r, LaneId l) noexcept;
std::optional<SignalView>      signal_of   (const Map& m, SignalId id)      noexcept;
std::optional<ObjectView>      object_of   (const Map& m, ObjectId id)      noexcept;
std::optional<JunctionView>    junction_of (const Map& m, JunctionId id)    noexcept;
std::optional<ControllerView>  controller_of(const Map& m, ControllerId id) noexcept;

// Fallible operations: std::expected<T, E> with per-op E
std::expected<Map, BuildError>   Map::from_file  (const std::filesystem::path&);
std::expected<Map, BuildError>   Map::from_buffer(std::span<const std::byte>);
std::expected<LocateResult, LocateError> locate(const QueryHandle&, double x, double y);
std::expected<Route, RouteError>         route (const QueryHandle&, std::span<const Waypoint>);

}  // namespace hodor
```

**Why these specific decisions:**

- **Lookups return `std::optional<View>`, not `std::expected<T*, LookupError>`.** "Not found" is a single failure mode; `optional` says exactly that. The `View` is a value type, not a `const T*`, because pointers in `expected`/`optional` are noisy at the call site (`if (auto r = road_of(...); r) { auto& x = **r; }` vs. `if (auto r = road_of(...); r) { auto& x = *r; }`).
- **Hot-path E is `enum class : std::uint8_t`.** A `std::expected<LocateResult, LocateError>` is small enough to live in two registers; the success path is `mov`-and-`test`, no branch for the payload. `LocateError` and `RouteError` are `uint8_t`-backed specifically for this.
- **Cold-path E is a struct with payload.** `BuildError` carries `xml_line`, `xml_column`, and a `std::string_view message` so the caller knows *where* the parse failed. The 32-byte cost is irrelevant in a one-shot build.
- **`std::string_view` message lifetime.** The view points into a string Hodor owns. The view is valid for the duration of the `std::expected<Map, BuildError>`'s lifetime, which is bounded by the build call. Consumers that need to keep the error beyond the call must copy the message. This is documented; the constraint is simple.
- **The `View` types are first-class.** They have methods (`road_view.id()`, `road_view.length()`, `road_view.lane_sections()`) and `std::span` accessors. They are not opaque handles; the caller can use them as if they were the entity.

**The implications worth recording explicitly:**

- The library does not use `try`/`catch` internally. C++ standard library components that throw (`std::vector::at`, `std::stoi`, `std::filesystem` copy operations in some implementations) are not used on the hot path. Cold path may use them internally and convert any throw into a `BuildError` via a small RAII guard (e.g., a `try { ... } catch (const std::exception& e) { return std::unexpected(BuildError{...}); }` at the build boundary, *not* sprinkled through internal code).
- Consumers do not wrap Hodor calls in `try`/`catch`. They pattern-match on the `expected`/`optional`.
- A future contributor proposing to add `throw` is wrong. The answer is "this is recorded in ADR 0005; please return a `std::expected` or `std::optional` instead."
- The CMake build sets `-fno-exceptions` (and the MSVC equivalent) on the `hodor` target. Consumers that link Hodor and use exceptions elsewhere are unaffected; the `-fno-exceptions` is local to Hodor's translation units.
