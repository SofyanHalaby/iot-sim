# ADR 0003: Build system — CMake + CMakePresets + vcpkg

**Status:** Accepted
**Date:** 2026-05-20

## Context

The project is C++20 ([ADR 0001](0001-language-and-standard.md)) with at least one external native dependency (Unicorn) and a likely-growing list as v0.2+ adds peripherals, ELF tooling, and test frameworks. v0.1 needs a build system decided in sprint 1 because every later sprint depends on it.

Constraints:

- IDE support: VS Code, CLion, Visual Studio (eventually) — they all speak CMake natively
- Reproducibility: a fresh clone should build with no manual dependency steps
- Cross-platform: Linux and macOS in v0.1; Windows in v0.2
- The embedded crowd is already CMake-fluent

## Decision

**CMake ≥ 3.25** with **`CMakePresets.json`** and **vcpkg in manifest mode** (`vcpkg.json`).

Presets at minimum: `debug`, `debug-asan`, `debug-ubsan`, `release`. The matrix of compiler × build type is expressed in CMakePresets, not in custom shell scripts.

Dependencies are pinned via a `vcpkg.json` manifest with a fixed `builtin-baseline`. vcpkg is added as a git submodule in `third_party/vcpkg/` (alternatively, fetched by `cmake/vcpkg-bootstrap.cmake` on first configure — decided in sprint 1).

## Consequences

**Easier:**
- One command for new contributors: `cmake --preset debug-asan && cmake --build --preset debug-asan`
- IDEs autoconfigure from `CMakePresets.json`
- Dependencies are reproducible without per-platform package-manager incantations
- Sanitizer toggles are just preset selection

**Harder:**
- CMake is verbose; we will write `cmake/*.cmake` helpers to keep `CMakeLists.txt` files terse
- vcpkg's baseline upgrades occasionally break ports — pin and upgrade deliberately, not auto

**Obligations on contributors:**
- All dependencies go in `vcpkg.json`; no system-package assumptions
- New presets are added to `CMakePresets.json`, not as `make` wrapper scripts
- CMake minimum stays at 3.25 — features above that require an ADR

## Alternatives considered

**Conan.** Comparable to vcpkg. Rejected because vcpkg's manifest mode + CMake integration is now smoother for the kinds of C/C++ libraries we'll consume (GoogleTest, fmt, Unicorn). Either would work.

**Bazel.** Rejected: foreign to the embedded C++ crowd, and the C++ rules ecosystem still has rough edges for vendored deps.

**Meson.** Nicer syntax but smaller ecosystem; less IDE support; smaller bus factor for contributors.

**Hand-rolled Makefiles.** Hard pass.

**Header-only / fetch-content for all deps.** Works for tiny deps but not for Unicorn or a Python binding library. We'd end up with two systems.
