# v0.1 Repository Layout

A concrete proposal for the directory and module structure once v0.1 implementation begins. The split exists to make the [ADR 0002](adrs/0002-cpu-backend-and-license-isolation.md) license boundary visible in the file tree and to keep peripherals additive (registry, not switch statements).

## Layout

```
iot-sim/
├── CMakeLists.txt
├── CMakePresets.json
├── vcpkg.json
├── .clang-format
├── .clang-tidy
├── cmake/
│   ├── compile_options.cmake     # warnings, -Werror, debug flags
│   ├── sanitizers.cmake          # ASan / UBSan toggles
│   └── unicorn.cmake             # builds Unicorn into the worker only
├── src/
│   ├── core/                     # logging, errors, config, types
│   │   ├── log.hpp / .cpp
│   │   ├── error.hpp
│   │   └── types.hpp
│   ├── elf/                      # ELF32-LE loader
│   │   ├── loader.hpp / .cpp
│   │   └── memory_map.hpp
│   ├── cpu/
│   │   ├── backend.hpp           # CpuBackend interface (host-side only)
│   │   ├── reg.hpp               # CoreReg enum, RunResult, Perms
│   │   └── unicorn/              # ── GPL ZONE ──
│   │       ├── worker_main.cpp   # the worker binary entrypoint
│   │       ├── worker.hpp / .cpp # Unicorn driver
│   │       └── subprocess_client.hpp / .cpp  # host-side client speaking IPC
│   ├── ipc/                      # neutral protocol (no Unicorn types leak)
│   │   ├── frame.hpp / .cpp
│   │   ├── kinds.hpp
│   │   └── socketpair.hpp / .cpp
│   ├── memory/
│   │   ├── map.hpp / .cpp        # region table
│   │   └── mmio_dispatcher.hpp / .cpp
│   ├── peripherals/
│   │   ├── peripheral.hpp        # base interface
│   │   ├── gpio/
│   │   │   ├── gpio.hpp / .cpp
│   │   │   └── README.md         # which registers, which simplifications
│   │   ├── systick/
│   │   │   └── systick.hpp / .cpp
│   │   └── rcc/
│   │       └── rcc.hpp / .cpp
│   ├── boards/
│   │   └── stm32f4_discovery/
│   │       ├── board.hpp / .cpp  # composes memory map + peripherals
│   │       └── README.md
│   └── cli/
│       ├── main.cpp              # iotsim entrypoint
│       └── args.hpp / .cpp
├── tests/
│   ├── CMakeLists.txt
│   ├── unit/
│   │   ├── elf/
│   │   ├── ipc/
│   │   ├── memory/
│   │   └── peripherals/
│   ├── integration/
│   │   ├── blinky_test.cpp
│   │   ├── systick_timing_test.cpp
│   │   └── mmio_smoke_test.cpp
│   └── fixtures/
│       ├── blinky.elf
│       ├── malformed/            # negative fixtures for the loader
│       └── src/                  # source for the fixtures (built via CMake)
├── examples/
│   └── blinky/
│       ├── Makefile
│       ├── README.md
│       └── src/
├── third_party/
│   └── unicorn/                  # vendored or fetched at configure time
├── docs/
│   ├── initial-plan/
│   └── v0.1/                     # this folder
└── .github/
    └── workflows/
        └── ci.yml
```

## Why this split

**`src/cpu/unicorn/` is the GPL zone.** Everything in it ends up only in `iotsim-cpu-worker`. Everything outside it ends up in `iotsim` (the host binary). CMake enforces this with two targets and a CI check (`make check-no-unicorn-in-host`). No header from `src/cpu/unicorn/` is allowed to be `#include`d from outside `src/cpu/unicorn/` or from the worker's own translation units — except `subprocess_client.hpp`, which is the host-side facade and contains no Unicorn types.

**`src/ipc/` is license-neutral.** It only knows about frames and bytes. Both the host and the worker link it. It is the only code physically shared across the license boundary.

**Peripherals are a registry, not a switch.** A board (`src/boards/.../board.cpp`) constructs concrete `Peripheral` instances and hands them to the `MmioDispatcher`. Adding GPIOB in v0.2 means a new file, no edits to the dispatcher.

**`tests/fixtures/src/` builds the fixture ELFs from source** via the GNU Arm Embedded Toolchain (or downloads pre-built artifacts if the toolchain isn't present). This keeps the repo reproducible without committing 50 KB binary blobs forever — the committed `.elf` files are a convenience for first-clone-no-toolchain users.

## Two CMake targets

```
add_executable(iotsim ${HOST_SOURCES})
target_link_libraries(iotsim PRIVATE core ipc memory peripherals cpu_host_facade cli)

add_executable(iotsim-cpu-worker ${WORKER_SOURCES})
target_link_libraries(iotsim-cpu-worker PRIVATE core ipc unicorn-static)
```

Only `iotsim-cpu-worker` is permitted to depend on Unicorn. The build will fail if you try to link `unicorn-static` into `iotsim`.

## Out of layout for v0.1

- `docs/v0.2/` and onward — will follow the same pattern as `docs/v0.1/`
- A Python package directory — lands in v0.4 when the pytest API ships
- A `bindings/` directory — lands when there's a second binding (probably v1.0+)
