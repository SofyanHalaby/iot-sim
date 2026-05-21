# ADR 0004: Test framework — GoogleTest + GoogleMock

**Status:** Accepted
**Date:** 2026-05-20

## Context

v0.1 has both fine-grained unit tests (ELF loader fields, IPC frame round-trips, MMIO dispatch math) and coarser integration tests (run a real firmware through the simulator and assert on output). Both flavors need to live in CI, run under sanitizers, and produce CTest-compatible output for IDE/CI integration.

## Decision

**GoogleTest 1.14+** as the unit test framework. **GoogleMock** for mocks (sparingly — most v0.1 testing is against real types, not mocks). Both via vcpkg.

Test executables are registered with CTest. CI invokes `ctest --output-on-failure --parallel`.

Integration tests live in `tests/integration/`, are GoogleTest binaries themselves, and use fixture ELFs from `tests/fixtures/`.

## Consequences

**Easier:**
- Dominant C++ test framework; new contributors don't need to learn a new one
- CLion / VS Code / Visual Studio all surface GTest results natively
- GoogleMock is available when we need it (peripheral interfaces in unit tests)
- Death tests for the loader's abort paths
- Parameterized tests for the fixture corpus

**Harder:**
- Macros are louder than Catch2's; `EXPECT_THAT(x, AllOf(...))` is verbose
- GoogleMock encourages over-mocking; we discipline this in code review

**Obligations on contributors:**
- Every new file under `src/` gets a corresponding `tests/unit/.../<file>_test.cpp` unless explicitly waived in PR
- Avoid mocking concrete types when the real implementation is cheap to construct (most of our types are)
- Integration tests must be deterministic — no wall-clock dependencies; use the simulation's instruction-count clock

## Alternatives considered

**Catch2.** Nicer macros (`REQUIRE(x == y)` reads great), smaller. Rejected because:
- GoogleTest's ecosystem (mocks, parameterized tests, death tests) is broader
- Build time is roughly the same now (Catch2 v3 split-header)
- IDE integration is slightly stronger for GoogleTest as of 2026
- Either would work; pick one and move on

**Doctest.** Faster compile, header-only. Rejected: smaller community, less IDE polish.

**Hand-rolled assertion macros.** Rejected. We're not here to build a test framework.
