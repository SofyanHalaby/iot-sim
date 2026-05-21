# v0.1 Technical Design

This document specifies *how* v0.1 is built. Scope and acceptance live in [scope.md](scope.md); decisions of record live in [adrs/](adrs/README.md).

---

## 1. Memory map

The STM32F407VG memory map for v0.1:

| Region | Base | Size | Permissions | Notes |
|---|---|---|---|---|
| Flash | `0x0800_0000` | 1 MiB | R+X | Firmware image loaded here |
| SRAM | `0x2000_0000` | 128 KiB | R+W | Stack, heap, .data, .bss |
| Peripherals | `0x4000_0000` | 512 MiB window | MMIO | Routed through the MMIO dispatcher |
| System (SCB / SysTick / NVIC) | `0xE000_0000` | 1 MiB window | MMIO | SysTick at `0xE000_E010` |

Anything outside these regions is unmapped. An access to unmapped memory aborts the simulation with a diagnostic.

The vector table is **aliased** at `0x0000_0000` to `0x0800_0000` for v0.1 (matches default BOOT0=0 behavior). No BOOT pin emulation.

## 2. ELF loader

**Input:** a single ELF32 little-endian file.

**Validation (strict, fail closed):**
- `EI_CLASS == ELFCLASS32`
- `EI_DATA == ELFDATA2LSB`
- `e_type == ET_EXEC`
- `e_machine == EM_ARM` (0x28)
- At least one `PT_LOAD` segment

**Loading:**
- Iterate `PT_LOAD` program headers.
- For each, validate that `[p_paddr, p_paddr + p_memsz)` falls entirely inside a declared writable-or-flash region.
- Copy `p_filesz` bytes into the target region; zero-fill the remaining `p_memsz - p_filesz` bytes (BSS).
- Reject overlapping segments.

**Reset vector handling:**
- After loading, read `initial_sp = mem32[0x0800_0000]` and `initial_pc = mem32[0x0800_0004]`.
- Clear bit 0 of `initial_pc` (Thumb-state bit) before setting it as PC; the Thumb mode is signaled to Unicorn via the CPU mode flag, not via the PC bit.
- Write `initial_sp` to `SP` (`R13`) before the first `run()`.

## 3. CPU integration

### 3.1 The `CpuBackend` interface

A single C++ header defines the backend contract. Unicorn-as-subprocess is one implementation; the interface is what the rest of the system depends on.

```cpp
class CpuBackend {
public:
    virtual ~CpuBackend() = default;

    // Memory
    virtual void map_region(uint32_t base, size_t size, Perms perms) = 0;
    virtual void write_mem(uint32_t addr, std::span<const uint8_t> data) = 0;
    virtual void read_mem(uint32_t addr, std::span<uint8_t> out) = 0;

    // Hooks
    virtual HookId hook_mmio(uint32_t base, uint32_t end, MmioHandler h) = 0;
    virtual void unhook(HookId id) = 0;

    // Registers
    virtual uint32_t read_reg(CoreReg r) = 0;
    virtual void write_reg(CoreReg r, uint32_t v) = 0;

    // Execution
    virtual RunResult run(uint64_t max_instructions, Duration max_wall) = 0;
    virtual RunResult step(uint32_t count = 1) = 0;
    virtual void raise_irq(uint32_t exception_number) = 0;  // reserved for v0.2
};
```

### 3.2 Unicorn subprocess implementation

The Unicorn-linked binary runs as a child process. The host (`iotsim`) talks to it over a Unix socket pair (created via `socketpair(AF_UNIX, SOCK_STREAM, 0)` before `fork+exec`). Rationale and license context: [ADR 0002](adrs/0002-cpu-backend-and-license-isolation.md).

- The worker binary, `iotsim-cpu-worker`, links Unicorn statically and is licensed GPL-2.0.
- The host (`iotsim`) does **not** link Unicorn and is free to take a different license for Pro features later.
- A short startup handshake validates protocol version.

### 3.3 Unicorn configuration

- Architecture: `UC_ARCH_ARM`, mode: `UC_MODE_THUMB | UC_MODE_MCLASS`.
- Hooks installed: `UC_HOOK_MEM_READ` and `UC_HOOK_MEM_WRITE` over the MMIO regions only; `UC_HOOK_MEM_UNMAPPED` globally; `UC_HOOK_INTR` (reserved for v0.2).
- Note: Unicorn does not model the Cortex-M exception entry/exit sequence. v0.1 does not need it — there are no interrupts beyond reset.

## 4. Peripheral framework

### 4.1 Dispatch

A single `MmioDispatcher` owns a sorted, non-overlapping table of `(base, end, Peripheral*)` tuples. On every MMIO hook callback from the CPU backend:

- Find the peripheral whose range contains the access (binary search).
- Call `peripheral->read32(offset)` or `write32(offset, value)`.
- If no peripheral matches, abort with `unmapped_mmio` diagnostic.

### 4.2 Peripheral interface

```cpp
class Peripheral {
public:
    virtual ~Peripheral() = default;
    virtual uint32_t read32(uint32_t offset) = 0;
    virtual void write32(uint32_t offset, uint32_t value) = 0;
    virtual void tick(uint64_t instructions_elapsed) {}  // optional
};
```

### 4.3 v0.1 peripherals

| Peripheral | Base | Registers implemented |
|---|---|---|
| **GPIOA** | `0x4002_0000` | MODER (mode bits for output detection), ODR (output data), BSRR (atomic set/reset). IDR returns the last value written to ODR. |
| **SysTick** | `0xE000_E010` | CTRL (ENABLE, TICKINT-read-only-as-0, COUNTFLAG), LOAD (RVR), VAL (CVR). |
| **RCC** | `0x4002_3800` | AHB1ENR (only bit 0 for GPIOAEN is honored; other bits stored but ignored). |

GPIO emits a state-change event whenever a previously-tracked pin changes value. Events are produced regardless of trace flags; the CLI decides whether to print them.

## 5. Time model

v0.1 uses an **instruction-count clock**, not cycle-accurate timing. The simulation has one knob:

```
instructions_per_us = 168   // configurable; default sized for STM32F4 @ 168 MHz typical IPC
```

`SysTick` countdown is driven from the host side: after each `run()` slice, the host asks the backend how many instructions ran, converts to microseconds, decrements `CVR`, and re-loads from `RVR` on underflow. This is documented as approximate and adequate for v0.1's blinky demonstration; cycle-accurate modeling is a separate effort (not in scope before v1.0).

## 6. CLI design

```
iotsim --version
iotsim run <firmware.elf> --board stm32f4-discovery
              [--max-time <duration>]      # default 10s simulated
              [--max-instructions <N>]     # safety cap; default 1e9
              [--trace gpio[,...]]
              [--output <file>]            # default stdout
```

**Output format** (newline-delimited, one event per line):

```
t=12.345ms gpio PA5 1
t=12.845ms gpio PA5 0
```

Exit codes: `0` clean, `1` user error (bad args, file not found), `2` simulation error (illegal instruction, unmapped memory), `3` time/instruction cap reached without firmware halting.

## 7. IPC protocol (host ↔ worker)

A minimal length-prefixed binary protocol over the Unix socket pair. **Not stable**; v0.1 only.

**Frame:**

```
u32 length      // little-endian, of (kind + payload)
u16 kind        // see table
u16 reserved    // 0
bytes payload
```

**Kinds (v0.1):**

| Kind | Direction | Payload |
|---|---|---|
| 0x0001 HELLO | host→worker | u32 protocol_version |
| 0x0002 HELLO_ACK | worker→host | u32 protocol_version |
| 0x0010 MAP | host→worker | u32 base, u32 size, u8 perms |
| 0x0011 WRITE_MEM | host→worker | u32 addr, u32 len, bytes |
| 0x0012 READ_MEM | host→worker | u32 addr, u32 len |
| 0x0013 READ_MEM_RESP | worker→host | bytes |
| 0x0020 HOOK_MMIO | host→worker | u32 base, u32 end |
| 0x0030 WRITE_REG | host→worker | u16 reg_id, u32 value |
| 0x0031 READ_REG | host→worker | u16 reg_id |
| 0x0032 READ_REG_RESP | worker→host | u32 value |
| 0x0040 RUN | host→worker | u64 max_instructions, u64 max_wall_ns |
| 0x0041 RUN_RESULT | worker→host | u8 reason, u64 instructions_executed |
| 0x0050 MMIO_EVENT | worker→host | u8 op (R/W), u32 addr, u32 value, u32 size |
| 0x0051 MMIO_REPLY | host→worker | u32 value |
| 0x00F0 SHUTDOWN | either way | (none) |

Hook callbacks are synchronous: when an MMIO event arrives, the worker is blocked until the host replies with the value (for reads) or an ACK (for writes). Throughput optimization is deferred; for v0.1 correctness comes first.

## 8. Logging and diagnostics

- `core/log.hpp` thin wrapper around `std::format` + level filter.
- Levels: `trace`, `debug`, `info`, `warn`, `error`.
- Default level: `info`. `--verbose` raises to `debug`; `-vv` to `trace`.
- All abort paths emit a structured one-line diagnostic before exiting (e.g. `error: unmapped read at 0x40004000 (pc=0x080001a4)`).

## 9. What's deliberately deferred

Things that will *look* missing to a knowledgeable reader and are not oversights:

- **Cortex-M exception model** — no NVIC, no preemption, no MSP/PSP switching. Lands in v0.2.
- **Cycle accuracy** — instruction-count clock is good enough for blinky. Real timing model lands when firmware needs it.
- **DMA, FPU, MPU** — out of scope.
- **Cross-platform** — Windows host deferred to v0.2 (subprocess + Unix socket pair are POSIX-shaped in v0.1).
