# ADR 0006: Initial MCU target — STM32F407VG / STM32F4-Discovery

**Status:** Accepted
**Date:** 2026-05-20

## Context

The master plan picks STM32F4 (Cortex-M4) as the first target ([line 42](../../initial-plan/embedded_simulator_release_plan.md)). v0.1 needs to pick a specific part within that family because the memory map, peripheral base addresses, and reset behavior are part-specific, not family-generic.

Constraints:

- Firmware samples need to be widely available so contributors and users have a baseline
- The reference part should be one that hobbyists already own physical hardware for (lets us validate against real silicon)
- Memory map and peripheral docs must be public and well-documented
- The part should *not* require us to implement features outside v0.1 scope (no Ethernet, no USB, no DCMI peripheral assumed by default-init code)

## Decision

**MCU:** STM32F407VG
**Reference board:** STM32F4-Discovery (eval board MB997C)

Memory map (used by [technical-design.md §1](../technical-design.md#1-memory-map)):
- Flash: `0x0800_0000`, 1 MiB
- SRAM: `0x2000_0000`, 128 KiB (CCM RAM at `0x1000_0000` deferred; not used by typical blinky)
- Peripheral window: `0x4000_0000`
- System window: `0xE000_0000`

The example firmware in `examples/blinky/` will be written for STM32F407VG, will toggle PA5 (a Discovery board LED variant) via SysTick polling, and will avoid the vendor HAL — bare CMSIS register access only.

## Consequences

**Easier:**
- Discovery board firmware is everywhere — ST examples, GitHub, blog posts
- The chip is in production and well-documented; ST's reference manual RM0090 is stable
- Hobbyist familiarity = adoption tailwind once v0.5 ships
- Vector table layout and reset behavior are standard ARMv7-M; nothing exotic

**Harder:**
- We are tying the v0.1 fixture firmware to ST headers (CMSIS device); we vendor those under `tests/fixtures/cmsis/` to keep the build reproducible
- Adding STM32F1 or STM32F7 in v1.5 requires touching `boards/`, `peripherals/` (different RCC layouts), and the memory map — that work is *planned*, not a regret

**Obligations on contributors:**
- v0.1 board code lives only in `src/boards/stm32f4_discovery/`. Generalization to other boards is v1.5 work and out of scope.
- The example firmware avoids the ST HAL — bare register access only, so v0.1 doesn't accidentally take on `SystemInit` complexity.

## Alternatives considered

**STM32F411 (Nucleo / BlackPill).** Smaller, also very common (BlackPill is a popular hobbyist board). Rejected because:
- The Discovery board is the canonical ST eval board cited in textbooks and tutorials
- F407 has more SRAM, leaving headroom for stack/heap in test firmwares
- Negligible difference in implementation effort either way; if F411 ends up easier we revisit

**Nordic nRF52.** Cortex-M4, similar surface. Rejected for v0.1 because:
- Vendor peripheral model is more idiosyncratic (EasyDMA, PPI) — would distract us from the blinky goal
- Deferred to v1.5 (BLE-focused)

**RP2040.** Dual Cortex-M0+, MIT-licensed SDK. Tempting but:
- M0+ does not implement all of Thumb-2 (no `IT` blocks, no UDIV/SDIV); good for a future minimum target but not where the majority of embedded codebases live
- Deferred to v1.5

**ESP32 (Xtensa).** Master plan v1.5 target. Out of scope for v0.1; not even ARM.

## Revisit triggers

This ADR should be re-evaluated if:

- The STM32F4-Discovery becomes unavailable or hard to source — choose Nucleo-F411RE as drop-in
- A specific paying customer needs a different first target as part of v1.0
