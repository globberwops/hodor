# SAX-style XML parse directly into the canonical Map; no OpenDRIVE validation

Hodor parses `.xodr` files. The 1.9 spec is XML. We have decided:

1. **SAX-style streaming parse directly into the canonical Map.** No intermediate DOM, no pugixml-style tree we then walk. The parser emits "start element / attribute / end element" events; a hand-written `OpenDriveBuilder` consumes those events and constructs the canonical `Map` in one pass. We use **Expat directly** (no third-party C++ wrapper) behind a ~30-line internal RAII shim. CMake: `find_package(EXPAT)` first, `FetchContent` of upstream Expat as the fallback.
2. **OpenDRIVE schema validation is out of scope.** The builder is **tolerant by design**: missing optional elements fall back to OpenDRIVE defaults, unknown elements are silently skipped, and out-of-order elements are best-effort. We *do* error on actual XML malformation (unclosed tags, bad attribute syntax, encoding errors) because those are unrecoverable.

The SAX choice is driven by the perf and dependency posture we already locked in:

- **No third-party DOM dependency** — we avoid pugixml and friends. SAX parsers like Expat are smaller, faster, and have a much thinner surface to integrate.
- **Lower peak memory** — no full document tree in memory at any point. For a 100 MB `.xodr` the difference between 2-3x file size (DOM) and ~2x the canonical Map (streaming) is meaningful.
- **The canonical Map is the only tree in the system.** No translation step, no double representation. Every byte goes from the SAX callback into the final field of the canonical `Map`.

The "no validation" decision is driven by the realities of real-world `.xodr` files: hand-edited maps, vendor-specific extensions, optional elements being absent, and OpenDRIVE 1.9 introducing new elements that older files don't have. Strict validation would mean refusing to load files that *are* usable, which is the opposite of what consumers want. The cost is that semantic errors (a `lane` referencing a non-existent `road`) are not caught at parse time; we accept that and treat the canonical `Map` as best-effort.

The implication worth recording explicitly: **there is no `validate_xodr()` API and there will not be one in v1.** If a future change wants schema validation, it is a separate project.
