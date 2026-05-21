# ADR 0005: Sanitizers and CI matrix

**Status:** Accepted
**Date:** 2026-05-20

## Context

We are writing C++20 ([ADR 0001](0001-language-and-standard.md)) and accepting the memory-safety burden that comes with it. The simulator loads developer-supplied firmware blobs through an ELF loader, which is a notorious source of OOB reads and buffer overflows in real-world tooling. We need the cheapest credible defense in depth from day one.

We also need the CI matrix decided before sprint 1 so contributors know what they're up against on PRs.

## Decision

**Sanitizers in CI (mandatory):**

- **AddressSanitizer (ASan)** — runs all unit + integration tests every PR
- **UndefinedBehaviorSanitizer (UBSan)** — runs all unit + integration tests every PR
- Sanitizers are enabled via CMake presets (`debug-asan`, `debug-ubsan`). Local devs can opt in.
- ThreadSanitizer (TSan) is **not** mandatory in v0.1 — host and worker each run single-threaded.

**Compiler flags (mandatory, all builds):**

```
-Wall -Wextra -Wpedantic -Werror
-Wshadow -Wnon-virtual-dtor -Wold-style-cast
-Wcast-align -Woverloaded-virtual -Wconversion
-Wsign-conversion -Wnull-dereference -Wdouble-promotion
-Wformat=2
```

Debug builds also add `-fno-omit-frame-pointer` and `-g3`.

**CI matrix (v0.1):**

| OS | Compiler | Build | Sanitizers |
|---|---|---|---|
| Linux x86_64 | gcc-13 | debug | ASan + UBSan |
| Linux x86_64 | clang-17 | debug | ASan + UBSan |
| Linux x86_64 | gcc-13 | release | — |
| macOS arm64 | Apple clang (Xcode 15+) | debug | ASan + UBSan |

Windows is **deferred to v0.2** (see [ADR 0002](0002-cpu-backend-and-license-isolation.md) consequences).

**Nightly CI (best-effort for v0.1):**

- libFuzzer harness over the ELF loader against `tests/fixtures/malformed/`
- Long-running blinky soak (1+ hour simulated)

## Consequences

**Easier:**
- Memory bugs are caught at PR time, not in production
- Contributors get fast feedback on UB they may not realize they wrote
- Sanitizer green is a real signal for v1.0 "fuzz-tested" claims

**Harder:**
- Sanitizer-instrumented tests are 2–5× slower; CI walltime grows. We accept it.
- Some Unicorn-internal allocations may trigger ASan false positives — suppression file in `cmake/asan.supp` if needed
- `-Werror` will block PRs that introduce warnings. Good. We will not relax this without an ADR.

**Obligations on contributors:**
- A PR is green only when both ASan and UBSan jobs pass
- New warnings are fixed at the source, not silenced
- Suppressions go in a tracked file with a comment explaining why

## Alternatives considered

**Sanitizers in nightly only, not per-PR.** Rejected: too easy to break and not notice until later. PR-time is the right gate.

**Drop UBSan, keep ASan.** Rejected: UBSan catches a different class of bugs (alignment, signed overflow, null derefs) very cheaply.

**Add MSan (MemorySanitizer).** Rejected for v0.1: requires instrumented libc/libc++; too much yak-shave. Reconsider when reaching v1.0 hardening pass.

**Add TSan.** Rejected for v0.1: no real concurrency yet. Add when v0.2+ introduces threads or async IPC.

**`-Wall -Wextra` without `-Wpedantic` and `-Werror`.** Rejected: warnings rot into the codebase. We'd rather pay the cost upfront and once.
