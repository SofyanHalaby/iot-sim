# v0.1 Sprint Plan

Six 2-week sprints, 12 weeks total, mapping to the Months 1–3 timeline in the master plan. Each sprint exits with something demonstrable; the gate to the next sprint is the **exit criteria**, not the calendar.

| Sprint | Theme | Demo at the end |
|---|---|---|
| 1 | Foundation | `iotsim --version` builds, CI green with sanitizers |
| 2 | Unicorn spike | Execute hand-assembled Thumb code, read a register back |
| 3 | ELF + memory | Load `blinky.elf` and reach `main()` |
| 4 | MMIO + GPIO | Firmware toggles PA5 (visible in trace) |
| 5 | SysTick + RCC | PA5 toggles at the *correct* frequency |
| 6 | Polish + ship | `make demo` runs blinky end-to-end; v0.1.0 tagged |

---

## Sprint 1 — Foundation (weeks 1–2)

**Goal:** A buildable, CI-tested C++ project skeleton with all the decisions of record landed before any feature code is written.

**Tasks:**

- Create the repo layout from [repo-layout.md](repo-layout.md): top-level dirs, empty `CMakeLists.txt` files
- Wire CMake ≥ 3.25 with `CMakePresets.json` (debug-asan, debug-ubsan, release)
- Adopt vcpkg in manifest mode; commit `vcpkg.json`
- Add GoogleTest as a vcpkg dependency; one passing dummy test
- CI workflow (`.github/workflows/ci.yml`): Linux gcc-13, Linux clang-17, macOS clang; build + ctest under ASan and UBSan
- Strict warnings: `-Wall -Wextra -Wpedantic -Werror`, `-fno-omit-frame-pointer` in debug
- Land ADRs 0001–0006 in `docs/v0.1/adrs/`
- Stub `iotsim` binary that prints version and exits 0
- `clang-format` config + a CI check

**Exit criteria:**
- `git clone && cmake --preset debug-asan && cmake --build` works on a clean Linux host
- CI passes on all three platforms
- `iotsim --version` prints something sensible
- All six ADRs are merged

**Risks:**
- vcpkg manifest mode + CMake presets occasionally fight each other on macOS — budget half a day
- Toolchain pinning vs. distro defaults; use vcpkg-installed clang-tidy if friction

---

## Sprint 2 — Unicorn spike (weeks 3–4)

**Goal:** Prove the subprocess + IPC seam works and Unicorn can run real instructions for us. No ELF, no peripherals — just registers and memory through the wire.

**Tasks:**

- Vendor or fetch Unicorn 2.x into `third_party/`; build it as part of the worker
- Build target `iotsim-cpu-worker`: links Unicorn, GPL-2.0 source header
- Build target `iotsim`: host process, does **not** link Unicorn
- Implement `socketpair` + `fork+exec` worker launcher in `src/ipc/`
- Implement the IPC frame layer (length-prefixed; see [technical-design.md §7](technical-design.md))
- Implement `HELLO/HELLO_ACK`, `MAP`, `WRITE_MEM`, `READ_MEM`, `READ_REG`, `WRITE_REG`, `RUN`, `RUN_RESULT`, `SHUTDOWN`
- Define `CpuBackend` interface header
- Implement `UnicornSubprocessBackend`
- **Spike test:** map 4 KiB SRAM, write hand-assembled Thumb (`movs r0, #42; bkpt`), run, read R0 → assert 42
- Measure RTT for a no-op IPC roundtrip; record in this doc

**Exit criteria:**
- Spike test passes under ASan
- A `make check-no-unicorn-in-host` rule confirms the host binary doesn't depend on Unicorn (e.g. `ldd iotsim | grep -v unicorn`)
- IPC roundtrip < 50µs on developer hardware (record actual)

**Risks:**
- Cortex-M mode flags (`UC_MODE_MCLASS`) — Unicorn version sensitivity; pin Unicorn version in `third_party/`
- Worker crash diagnostics: host needs to detect `SIGCHLD` and surface a useful message

---

## Sprint 3 — ELF + memory (weeks 5–6)

**Goal:** Load a real `blinky.elf` into memory and execute up to the first MMIO access. The firmware is allowed to fault when it touches GPIO — that's sprint 4's problem.

**Tasks:**

- Implement `src/elf/` loader per [technical-design.md §2](technical-design.md)
- Add fixture ELFs in `tests/fixtures/`: minimal valid, missing segments, overlapping segments, wrong machine, segments outside memory map
- Implement `src/memory/` memory-map module
- Wire `iotsim run` to: load ELF → map regions → write image → set SP/PC from reset vector → run until unmapped MMIO
- Integration test: a hand-built fixture firmware that loops, runs for N instructions, host stops it

**Exit criteria:**
- Loader rejects all five negative-fixture ELFs with distinct errors
- `iotsim run blinky.elf` executes ≥ 1000 instructions before hitting unmapped MMIO (proves reset vector + Thumb mode + branches all work)
- ASan + UBSan clean on the loader fuzz corpus (use a small `libFuzzer` harness if time allows)

**Risks:**
- Reset-vector Thumb-bit confusion; double-check Unicorn behavior on real STM32F4 firmware vs. spec
- ELF endianness handling in C++ structured bindings — write small unit tests for each header parse

---

## Sprint 4 — MMIO dispatch + GPIO (weeks 7–8)

**Goal:** Real firmware toggles a GPIO pin and the trace shows it.

**Tasks:**

- Implement `src/memory/MmioDispatcher` (sorted ranges, binary search lookup, abort on unmapped)
- Implement `Peripheral` interface
- Implement `peripherals/gpio/` for GPIOA: MODER, ODR, BSRR. IDR mirrors ODR.
- GPIO state-change event emission to the host log channel
- `--trace gpio` CLI flag and the `t=…ms gpio PAn 0|1` output format
- Wire it all together in `boards/stm32f4_discovery/`
- Integration test: a tiny firmware that writes a known pattern to BSRR, asserts the produced trace exactly

**Exit criteria:**
- `iotsim run blinky.elf --trace gpio` prints PA-line events
- Toggling via BSRR atomic semantics is correct (set has priority over reset within the same write — verify against ARM RM)

**Risks:**
- GPIO MODER decoding subtleties; v0.1 simplifies — we trust ODR/BSRR even if MODER says input. Document that trade-off.

---

## Sprint 5 — SysTick + RCC (weeks 9–10)

**Goal:** The blinky frequency is *correct*, not just present.

**Tasks:**

- Implement `peripherals/systick/`: CTRL, LOAD, VAL. Countdown decremented from the host on each `RUN` slice.
- Implement `peripherals/rcc/` (minimal): AHB1ENR with GPIOAEN bit honored, everything else stored.
- Add the instruction-count clock and `instructions_per_us` knob; default 168.
- Integration test: a fixture firmware that uses SysTick to toggle PA5 every 500 ms; assert the trace's edge timestamps fall within ±10%.
- README updates inside `examples/blinky/` explaining how to interpret trace output

**Exit criteria:**
- The acceptance criterion #2 in [scope.md](scope.md) passes for the canonical blinky firmware

**Risks:**
- The `instructions_per_us` heuristic will be wrong for some firmwares; document it as a knob and call it out as an explicit v0.1 limitation

---

## Sprint 6 — Polish + ship (weeks 11–12)

**Goal:** A new user can clone the repo and have blinky running within 5 minutes.

**Tasks:**

- Build the `examples/blinky/` project end-to-end, including firmware source and a `Makefile` that produces `blinky.elf` using the GNU Arm Embedded Toolchain
- A top-level `make demo` that builds firmware → builds iotsim → runs it → shows trace
- Error message audit: every failure path emits a one-line, actionable message
- README on the project root pointing to `docs/v0.1/`
- Tag `v0.1.0`; produce binary release artifacts for Linux x86_64 and macOS (arm64) via CI
- Write a 1-page "how this works" doc for the project README

**Exit criteria:**
- All 7 items in [scope.md acceptance criteria](scope.md#acceptance-criteria) pass
- `git tag v0.1.0` pushed, GitHub release with binaries attached
- The 5-minute onboarding (from the master plan v0.5 goal) is achievable for v0.1 at the CLI level

**Risks:**
- GNU Arm Embedded Toolchain install friction on macOS arm64 — provide a Homebrew tap instruction or a Docker fallback
