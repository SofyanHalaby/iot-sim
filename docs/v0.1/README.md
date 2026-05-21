# iot-sim v0.1 — "Hello World"

**Status:** Planning · **Timeline:** Months 1–3 · **Goal:** Boot and run a real STM32F4 `blinky.elf` end-to-end via the CLI.

This folder is the source of truth for v0.1 execution. The [master release plan](../initial-plan/embedded_simulator_release_plan.md) covers the whole roadmap; everything specific to v0.1 lives here.

## Contents

| Doc | What it covers |
|---|---|
| [scope.md](scope.md) | Goal, in-scope / out-of-scope, acceptance criteria, definition of done |
| [technical-design.md](technical-design.md) | Memory map, ELF loader, CPU integration, peripherals, time model, CLI, IPC |
| [sprints.md](sprints.md) | Six 2-week sprints with tasks, exit criteria, risks |
| [repo-layout.md](repo-layout.md) | Proposed folder and module structure |
| [risks.md](risks.md) | v0.1 risk register with mitigations |
| [adrs/](adrs/README.md) | Architecture Decision Records |

## At a glance

- **Stack:** C++20 · CMake + vcpkg · GoogleTest · Unicorn Engine 2.x (as a subprocess for GPL isolation)
- **Target board:** STM32F4-Discovery (STM32F407VG, Cortex-M4)
- **In scope:** ELF loading, Flash + SRAM, GPIO, SysTick, minimal RCC, CLI
- **Out of scope (deferred to v0.2):** UART, NVIC / interrupts beyond reset, debug protocol, real-time pacing
- **Done means:** `iotsim run blinky.elf --board stm32f4-discovery` toggles a GPIO pin at the expected frequency, runs 10+ seconds clean, passes CI with ASan + UBSan
