# v0.1 Risk Register

Risks specific to v0.1. Project-wide risks live in the [master plan](../initial-plan/embedded_simulator_release_plan.md#-risk-register). When a master-plan risk applies to v0.1, it appears here with v0.1-specific mitigations.

Each entry: **Likelihood** (Low / Medium / High) × **Impact** (Low / Medium / High) → **Mitigation**.

## Carried from the master plan, scoped to v0.1

### CPU simulation accuracy issues
**L:** High · **I:** High
Unicorn does not model the Cortex-M exception system. v0.1 sidesteps this by keeping interrupts out of scope, but firmware that *expects* interrupts to fire (e.g. uses SysTick interrupt instead of polling COUNTFLAG) will not work.
**Mitigation:** Document the constraint loudly in `examples/blinky/README.md`. Choose a blinky firmware that polls SysTick `COUNTFLAG` rather than relying on the SysTick interrupt. Defer interrupt-driven firmware to v0.2.

### Performance not good enough
**L:** Medium · **I:** Medium
IPC roundtrip cost per MMIO access could dominate runtime if firmware hits MMIO frequently (e.g. busy-polling registers).
**Mitigation:** Measure during sprint 2. If RTT > 50µs, batch MMIO read responses or fall back to in-process Unicorn for the community edition (the host can be GPL there). Decision point in sprint 4.

### Scope creep
**L:** High · **I:** Medium
The temptation to add UART, NVIC, "just one more peripheral" before blinky works.
**Mitigation:** [scope.md](scope.md) is the bar; anything not listed there is a v0.2 PR. The sprints exit on demo, not on "feature feels done."

## New, specific to v0.1

### GPL contamination of the host binary
**L:** Medium · **I:** High
A sloppy include, a misnamed CMake target, or a future contributor moves a Unicorn-touching file into shared code, and suddenly `iotsim` links Unicorn. The open-core strategy breaks silently.
**Mitigation:**
1. Two distinct CMake targets, only `iotsim-cpu-worker` is allowed to link `unicorn-static` (build system enforced).
2. CI step `make check-no-unicorn-in-host` that runs `ldd iotsim` (Linux) / `otool -L iotsim` (macOS) and greps for unicorn — fails the build if found.
3. ADR 0002 lists the rule and the reason; every reviewer can cite it.

### Cortex-M exception model gaps surface during blinky
**L:** Medium · **I:** Medium
A blinky firmware compiled with a vendor HAL might depend on `SystemInit` doing things v0.1 hasn't modeled (clock configuration, vector table relocation).
**Mitigation:** Provide a *known-good* blinky firmware in `examples/blinky/` that uses bare registers, no HAL. Vendor-HAL compatibility is a stretch goal for sprint 6, not a requirement.

### IPC protocol churn during early sprints
**L:** High · **I:** Low
The frame protocol in [technical-design.md §7](technical-design.md) will need changes once we feel the shape of MMIO callbacks. If host and worker drift, debugging is painful.
**Mitigation:** `HELLO` carries a version number; mismatched versions exit immediately with a useful error. Bump the version on every change. Single repo, single PR for protocol changes — host and worker move atomically.

### ELF loader edge cases (security and correctness)
**L:** Medium · **I:** Medium
Malformed ELFs are an obvious vector for buffer overflows and OOB reads. Even with ASan we want defense in depth.
**Mitigation:**
1. Validate aggressively in the loader (every field, every range, every overlap).
2. Commit a fixture corpus of malformed ELFs to `tests/fixtures/malformed/`.
3. Run a small libFuzzer harness in CI nightly (best-effort for v0.1; required for v1.0).

### Reset-vector Thumb-bit handling
**L:** Medium · **I:** Medium
Cortex-M reset vectors store PC with bit 0 set (Thumb marker). Forgetting to clear it before passing to Unicorn produces silent misbehavior (off-by-one PC, unaligned fetch).
**Mitigation:** Explicit unit test for the loader's reset-vector path with both correct and intentionally-malformed reset vectors. Document the rule in the loader header.

### Toolchain / vcpkg flakiness on macOS arm64
**L:** Medium · **I:** Low
Sprint 1 setup pain on Apple Silicon often eats more time than expected (Unicorn ARM64 builds, vcpkg port availability).
**Mitigation:** Pin a specific vcpkg baseline in `vcpkg.json`. Build the dev-loop container image (Linux) early so contributors can fall back to it if their host is hostile.

### GNU Arm Embedded Toolchain install friction blocks new users in sprint 6
**L:** Medium · **I:** Low
The 5-minute onboarding goal becomes 45 minutes if `arm-none-eabi-gcc` is hard to install.
**Mitigation:** Ship a pre-built `blinky.elf` in `tests/fixtures/` so the *demo* doesn't depend on the toolchain. The toolchain is only needed to *modify* the example firmware. Document a Docker fallback.

### Unicorn 2.x upstream changes
**L:** Low · **I:** Medium
Pinning Unicorn to a specific tag means we miss fixes; not pinning means we break unpredictably.
**Mitigation:** Pin to a specific Unicorn tag in `third_party/unicorn.cmake`. Re-evaluate at the end of each release. Track Unicorn upstream in a separate issue, not in v0.1 sprints.
