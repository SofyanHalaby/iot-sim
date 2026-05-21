# iot-sim

A developer-first embedded systems simulator for testing real firmware binaries without physical hardware.

## What it is

iot-sim aims to close the gap between QEMU (powerful but complex) and commercial tools (closed and dated) by offering a modern simulation platform that:

- Runs real firmware binaries on emulated MCUs (starting with STM32F4 / Cortex-M4)
- Models common peripherals (GPIO, UART, SPI, I2C, ADC, timers, interrupts)
- Provides behavioral models for external sensors and actuators
- Integrates into pytest and CI/CD workflows
- Supports fault injection for resilience testing

## Status

Early planning. No code yet.

- **Active milestone:** v0.1 — boot a real STM32F4 `blinky.elf` end-to-end. See [`docs/v0.1/README.md`](docs/v0.1/README.md) for scope, technical design, sprints, ADRs, and risks.
- **Full roadmap (v0.1 → v2.0):** [`docs/initial-plan/embedded_simulator_release_plan.md`](docs/initial-plan/embedded_simulator_release_plan.md).

## Target users

- Firmware developers (IoT, robotics, automotive)
- Embedded teams building CI/CD test automation
- Developers blocked on physical hardware
- Educators teaching embedded systems

## Technical direction

| Layer | Choice |
|---|---|
| CPU emulation | Unicorn Engine |
| Core language | Rust or C++ |
| Scripting / test API | Python |
| First target MCU | STM32F4 (Cortex-M4) |
| Test framework | pytest |
| License (community) | Apache 2.0 / MIT (TBD) |

## Roadmap at a glance

| Release | Theme |
|---|---|
| v0.1 | Boot firmware, GPIO, SysTick |
| v0.2 | UART, interrupts, timers, debug |
| v0.3 | SPI/I2C, sensors, actuators |
| v0.4 | Python test API, pytest plugin, fault injection |
| v0.5 | Public beta |
| v1.0 | GA + first Pro features |
| v1.5 | Multi-MCU (nRF52, RP2040, ESP32, RISC-V) |
| v2.0 | Cloud platform, enterprise |

## Repository layout

```
.
├── docs/                 # Planning and design documents
│   └── initial-plan/
└── README.md
```

## License

To be finalized. Community edition will use a permissive license (Apache 2.0 or MIT); Pro/Enterprise features will be commercially licensed under an open-core model.
