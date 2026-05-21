# ADR 0001: Language and standard — C++20

**Status:** Accepted
**Date:** 2026-05-20

## Context

The master plan ([line 41](../../initial-plan/embedded_simulator_release_plan.md)) lists "Rust or C++" as candidates for the core language. v0.1 needs the question settled before sprint 1 begins, because it determines build system, dependency manager, and contributor profile from day one.

Constraints in play:

- Unicorn Engine is a C library — the host language has to interop with C cleanly.
- The target audience (embedded firmware developers) is overwhelmingly C/C++.
- The Python test API (lands in v0.4) needs an idiomatic FFI from the core.
- The simulator loads developer-supplied firmware blobs — memory safety in the host matters.
- The team is small; tooling friction must be low.

## Decision

**C++20.** Specifically: C++20, with the following standard-library features expected to be available across our CI matrix: `std::span`, `std::format`, designated initializers, concepts, and ranges. `std::format` falls back to `fmt` via vcpkg where compiler support is incomplete.

C++23 is **not** adopted; standard-library support is still uneven across gcc-13 / clang-17 / Apple clang as of 2026-05.

## Consequences

**Easier:**
- Direct, no-FFI integration with Unicorn (C library → C++ host)
- pybind11 gives a first-class Python binding when the test API lands (v0.4)
- Familiar tooling for the embedded audience: gdb, valgrind, AddressSanitizer
- Large pool of contributors who can read and write the code

**Harder:**
- Memory-safety burden falls on us; mitigated by [ADR 0005](0005-sanitizers-and-ci.md) (ASan + UBSan mandatory in CI) and a "no raw `new`/`delete`, prefer `std::unique_ptr` and `std::span`" code-review rule
- Build complexity is real; mitigated by [ADR 0003](0003-build-system.md)
- C++20 modules adoption deferred — too many toolchain wrinkles in 2026

**Obligations on contributors:**
- All host-side code uses RAII; no raw owning pointers
- All buffer/range APIs take `std::span`, not `(ptr, size)` pairs
- Code compiles clean under `-Wall -Wextra -Wpedantic -Werror`

## Alternatives considered

**Rust.** Strong memory-safety story, modern tooling. Rejected because:
- Unicorn FFI adds maintenance burden (binding crates lag upstream)
- Audience fit weaker; fewer embedded firmware developers read Rust idiomatically in 2026
- pybind11/PyO3 are roughly equivalent for the Python binding, so that doesn't tilt the choice
- Cargo vs. CMake-for-the-embedded-world is a wash

Rust remains a viable rewrite target post-v1.0 if the CPU backend abstraction proves clean.

**C++17.** Considered for compiler portability. Rejected because:
- `std::span` is the right type for memory APIs; back-porting from `gsl::span` is friction we don't need
- Designated initializers (used heavily for config structs) require C++20
- gcc-13 and clang-17 both ship strong C++20 support; portability cost is minimal

**C (no C++).** Rejected: too much manual lifetime tracking for a project that will grow features quickly.
