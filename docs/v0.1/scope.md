# v0.1 Scope

## Goal

Boot a real STM32F4 firmware binary (`blinky.elf`) on the simulator and observe a GPIO pin toggle at the expected frequency, driven entirely by the firmware's own SysTick loop. Everything else is in service of this single end-to-end demonstration.

## In scope

- **CPU:** ARM Cortex-M4 (Thumb-2) via Unicorn Engine 2.x, isolated in a subprocess
- **Target board:** STM32F4-Discovery (MCU: STM32F407VG)
- **Memory regions:** Flash (read+execute), SRAM (read+write), peripheral MMIO window
- **ELF loader:** ELF32 little-endian, `EM_ARM`, `PT_LOAD` segments only
- **Reset handling:** read initial SP and PC from the Cortex-M reset vector
- **Peripherals (minimum viable):**
  - **GPIO** — port A only (configuration, output data register, pin-state tracking)
  - **SysTick** — RVR, CVR, CSR; countdown driven by the simulation's time model
  - **RCC** — only what's needed to enable the GPIOA clock
- **CLI:** `iotsim run <firmware.elf> --board stm32f4-discovery`, with `--max-time`, `--trace gpio`
- **Output:** GPIO state-change events to stdout in a parseable form (see [technical-design.md](technical-design.md))

## Explicitly out of scope (deferred to v0.2 or later)

- UART / USART (any direction)
- NVIC and interrupts beyond reset
- EXTI, DMA, timers other than SysTick
- All other GPIO ports (B–K)
- Floating point (VFP) support
- Cycle-accurate timing
- Debug protocol (gdbserver), trace, semihosting
- Windows host support
- Non-STM32 boards
- Python test API (lands in v0.4)

## Acceptance criteria

A v0.1 release is shippable when **all** of the following are true:

1. `iotsim run blinky.elf --board stm32f4-discovery --max-time 10s` exits cleanly with code 0.
2. Trace output shows a target GPIO pin toggling at the firmware's intended frequency (±10%).
3. Firmware runs for 10+ simulated seconds without a crash, illegal-instruction, or unmapped-memory event.
4. All unit and integration tests pass on Linux (gcc-13, clang-17) and macOS (clang).
5. CI runs the integration test under AddressSanitizer **and** UndefinedBehaviorSanitizer, clean.
6. Documentation in `docs/v0.1/` is current and links resolve.
7. An `examples/blinky/` project builds firmware from source (with the GNU Arm Embedded Toolchain) and runs it under iotsim in one `make demo` invocation.

## Definition of done (per feature within v0.1)

A feature inside v0.1 is *done* when:

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests written and passing
- [ ] At least one integration-level usage (real firmware or a fixture ELF)
- [ ] ASan + UBSan clean on the relevant tests
- [ ] Public API documented inline; user-facing behavior documented in `docs/v0.1/`
- [ ] No new compiler warnings under `-Wall -Wextra -Wpedantic -Werror`
