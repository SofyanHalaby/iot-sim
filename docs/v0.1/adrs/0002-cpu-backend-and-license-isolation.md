# ADR 0002: CPU backend — Unicorn 2.x in a subprocess for GPL isolation

**Status:** Accepted
**Date:** 2026-05-20

## Context

The master plan picks Unicorn Engine for CPU emulation ([line 39](../../initial-plan/embedded_simulator_release_plan.md)) on the grounds that writing a Cortex-M emulator from scratch is a year-plus of work before any firmware boots. That reasoning stands.

However, the master plan mislabels Unicorn as LGPL. Unicorn 2.x is **GPL-2.0** (verified at https://github.com/unicorn-engine/unicorn). This matters because:

- The master plan also commits to an **open-core model** ([line 44](../../initial-plan/embedded_simulator_release_plan.md), [line 370 onward](../../initial-plan/embedded_simulator_release_plan.md)), where the community edition is permissive (Apache 2.0 / MIT) and Pro features ship under a commercial license.
- A binary that statically or dynamically links a GPL-2.0 library inherits GPL-2.0 obligations on the combined work. Pro features that link Unicorn must be GPL-2.0 — which is incompatible with the commercial intent.

We considered four options:

1. Drop Unicorn, write our own CPU emulator
2. Adopt a permissive emulator (Renode-as-library, or rolling Thumbulator)
3. Accept that everything is GPL-2.0 and abandon open-core
4. Keep Unicorn but isolate it behind a process boundary

Option 1 costs 12+ months and contradicts the master plan's principle of "use existing tech where possible." Option 2 has no mature permissive equivalent for Cortex-M emulation. Option 3 throws away the monetization strategy. Option 4 is well-trodden (GDB-as-a-subprocess pattern; many fuzzing toolchains).

## Decision

**Use Unicorn Engine 2.x as the v0.1 CPU backend, run as a child process behind a stable `CpuBackend` C++ interface.** The host binary (`iotsim`) does not link Unicorn. A separate worker binary (`iotsim-cpu-worker`) links Unicorn and exposes the CPU through a binary IPC protocol over a Unix socket pair (see [technical-design.md §7](../technical-design.md#7-ipc-protocol-host--worker)).

The license boundary is:

- `iotsim` (host): permissive license (Apache 2.0 or MIT, chosen at v0.5)
- `iotsim-cpu-worker`: GPL-2.0 (inherits from Unicorn)
- `src/ipc/`: license-neutral; the only code shared across the boundary

Two CMake targets, enforced separation: see [repo-layout.md](../repo-layout.md) for the layout and `make check-no-unicorn-in-host` for the CI enforcement.

## Consequences

**Easier:**
- Pro features can ship under whatever license we want; they live in the host binary or as host-side plugins
- A future CPU backend (custom emulator, Renode, paid backend) plugs in behind `CpuBackend` without touching the rest of the system
- License story is explainable to a lawyer in one diagram

**Harder:**
- IPC overhead per MMIO callback — measured in sprint 2. Goal: ≤ 50 µs roundtrip on developer hardware. If MMIO-heavy firmware proves too slow, batch responses (sprint 4 decision point).
- Two binaries to ship and version together. Mitigated by a protocol-version handshake on `HELLO`.
- Debugging a crash in the worker requires attaching gdb to a child process. CI surfaces worker stderr in the host log.
- Windows host support deferred (no Unix socket pair). The worker would use Windows named pipes; treated as a v0.2 task.

**Obligations on contributors:**
- Never `#include` anything from `src/cpu/unicorn/` outside the worker binary, except `subprocess_client.hpp`.
- Never add Unicorn to the host CMake target. The build will refuse, but reviewers should also catch attempts in PR.
- Any changes to the IPC protocol bump the version in `HELLO` and must be made in the same PR for host and worker.

## Alternatives considered

**Drop Unicorn, write our own CPU emulator.** Rejected: 12+ months before any firmware boots, kills the time-to-market thesis, and the differentiated value is what's built *on top* of the CPU. Reconsider after v1.5 if there's a strategic reason.

**Accept GPL-2.0 across the entire product.** Rejected: kills the open-core revenue model in [the master plan](../../initial-plan/embedded_simulator_release_plan.md#-monetization-strategy). The whole Pro / Enterprise tier needs proprietary licensing.

**Renode-as-library.** Renode is MIT but is a much heavier framework (full board modeling, its own RESC scripting). Embedding it would import a lot of design we want to do our own way. Reconsider if v1.5 needs multi-MCU and Renode covers the targets cheaply.

**LGPL workaround via dynamic linking.** Unicorn is GPL-2.0, not LGPL-2.0. Dynamic linking does not unbind us from GPL obligations. Not a real alternative.

## Revisit triggers

This ADR should be re-evaluated if any of the following becomes true:

- IPC overhead measurably caps simulation throughput below the master plan's "50% realtime" performance target
- Unicorn relicenses (very unlikely but cheap to recheck annually)
- A permissively-licensed Cortex-M emulator with comparable maturity appears
- We decide to write our own backend (e.g., a paid customer funds it)
