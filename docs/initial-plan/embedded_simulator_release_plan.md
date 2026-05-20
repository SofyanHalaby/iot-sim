# Embedded Systems Simulator — Agile Release Plan

> A developer-first simulation platform for testing embedded firmware without physical hardware.

---

## 🎯 Product Vision

Build a **modern, developer-friendly embedded systems simulator** that runs real firmware binaries, enables fault injection, and integrates seamlessly into CI/CD pipelines — closing the gap left by QEMU (too complex) and commercial tools (closed and outdated).

### Target Users
- Firmware developers (IoT, robotics, automotive)
- Embedded teams needing CI/CD test automation
- Hardware-blocked developers (supply chain, remote work)
- Educational institutions teaching embedded systems

### Success Metrics
- ⏱️ Run real firmware binary within 5 minutes of install
- 🧪 Enable pytest-style firmware testing
- 🔄 Reduce hardware-in-loop testing time by 70%+
- 💰 Convert 5% of free users to paid tier within 12 months

---

## 📐 Guiding Principles

1. **Developer experience first** — fight QEMU's complexity at every turn
2. **Start narrow, expand wisely** — one MCU family done well > five done poorly
3. **Behavior over physics** — simulate what firmware sees, not real-world physics
4. **Build in public** — early users shape the product
5. **Use existing tech where possible** — Unicorn Engine for CPU, not custom

---

## 🏗️ Technical Stack (Initial)

| Layer | Choice | Rationale |
|---|---|---|
| CPU emulation | **Unicorn Engine** | LGPL, battle-tested, saves 6-12 months |
| Core language | **Rust** or **C++** | Performance for simulation loop |
| API/Scripting | **Python** | Where developers will write tests |
| First target MCU | **STM32F4 (Cortex-M4)** | Huge user base, well documented |
| Test framework | **pytest integration** | Familiar to developers |
| License (community) | **Apache 2.0** or **MIT** | Permissive, commercial-friendly |
| License (pro) | **Commercial** | Open core model |

---

## 🚦 Release Roadmap Overview

```
v0.1 ─ v0.2 ─ v0.3 ─ v0.4 ─ v0.5 ─── v1.0 ─── v1.5 ─── v2.0
 │     │     │     │     │      │       │       │
 MVP   I/O   Real  Test  Beta   GA      Multi   Enter-
       Devs  Sens  API   Users  Launch  MCU     prise
```

| Release | Timeline | Theme |
|---|---|---|
| **v0.1** | Month 1-3 | Internal MVP — boot firmware |
| **v0.2** | Month 4-5 | Core peripherals working |
| **v0.3** | Month 6-7 | External sensors & actuators |
| **v0.4** | Month 8-9 | Test API & developer experience |
| **v0.5** | Month 10-12 | Public beta |
| **v1.0** | Month 13-14 | General Availability |
| **v1.5** | Month 15-18 | Multi-MCU support |
| **v2.0** | Month 19-24 | Enterprise & cloud |

---

## 📦 Release v0.1 — "Hello World" (Months 1-3)

**Goal:** Run a simple "blinky" firmware binary end-to-end.

### Epic 1: CPU Foundation
- Integrate Unicorn Engine for ARM Cortex-M4
- Memory regions: Flash (read-only), SRAM (read/write)
- ELF file loader (parse and load firmware binary)
- Reset vector handling

### Epic 2: Minimum Peripherals
- GPIO peripheral (input/output, pin state tracking)
- SysTick timer (essential for any firmware loop)
- Basic clock/RCC (just enough to enable peripherals)

### Epic 3: Developer CLI
- Command-line tool to load and run firmware
- Print GPIO state changes to console
- Stop/start/step controls

### Acceptance Criteria
- ✅ Load STM32F4 blinky.elf file
- ✅ Observe GPIO pin toggling at expected frequency
- ✅ Firmware runs for 10+ seconds without crashes

### Risks
- Unicorn Engine learning curve
- ELF parsing edge cases
- Clock timing accuracy

---

## 📦 Release v0.2 — "Serial Debug" (Months 4-5)

**Goal:** Run more complex firmware with peripheral interaction.

### Epic 1: Communication Peripherals
- UART/USART (transmit only initially)
- UART receive with interrupt support
- Print firmware UART output to console

### Epic 2: Interrupt System
- NVIC implementation
- Interrupt priorities and preemption
- External interrupt (EXTI) for GPIO

### Epic 3: Additional Timers
- General-purpose timers (TIM2, TIM3)
- PWM output generation
- Input capture mode

### Epic 4: Debug Infrastructure
- Memory inspector (read state at any address)
- Register dump on demand
- Execution trace logging

### Acceptance Criteria
- ✅ Run FreeRTOS blinky example
- ✅ Capture UART debug output
- ✅ Generate PWM at correct frequency
- ✅ Handle external interrupts from simulated button press

---

## 📦 Release v0.3 — "Connected World" (Months 6-7)

**Goal:** Enable firmware to interact with simulated external devices.

### Epic 1: SPI & I2C
- SPI master mode (modes 0-3)
- I2C master mode (addressing, ACK/NACK)
- Bus-level transaction modeling

### Epic 2: ADC & DAC
- ADC with injectable values
- Multi-channel scanning
- DMA mode for ADC

### Epic 3: Sensor Models (Behavioral)
- BME280 (temperature/humidity/pressure)
- MPU6050 (IMU)
- DS18B20 (1-Wire temperature)
- Generic I2C device template

### Epic 4: Actuator Feedback
- DC motor model (PWM → encoder pulses)
- Servo model (PWM → position)
- Stepper motor model (step/dir → position)

### Acceptance Criteria
- ✅ Read realistic sensor data over I2C from firmware perspective
- ✅ Motor encoder feedback matches PWM input
- ✅ Run a complete IoT-style application (sensor → process → UART output)

---

## 📦 Release v0.4 — "Test Like a Pro" (Months 8-9)

**Goal:** Make the simulator pleasant to use for real testing workflows.

### Epic 1: Python Test API
```python
sim = Simulator(firmware="blink.elf", board="stm32f4")
sim.gpio("PA5").assert_high()
sim.sensor("BME280").set_temperature(42)
sim.advance_time(ms=500)
```

### Epic 2: Pytest Integration
- pytest plugin for embedded testing
- Fixtures for common setup
- Assertions for hardware state

### Epic 3: Fault Injection Framework
- Sensor disconnect / timeout
- Corrupted data / bad CRC
- Power brownout simulation
- I2C bus stuck low

### Epic 4: Watchdog & Power
- Watchdog timer with reset behavior
- Sleep modes (light, deep, standby)
- Wake-up source handling

### Acceptance Criteria
- ✅ Write a complete test suite in pytest
- ✅ Inject sensor faults and verify firmware error handling
- ✅ Test sleep/wake cycles correctly

---

## 📦 Release v0.5 — "Public Beta" (Months 10-12)

**Goal:** Launch publicly, gather feedback, iterate.

### Epic 1: Documentation
- Getting started guide (< 5 min to first run)
- Complete API reference
- 10+ working example projects
- Video walkthroughs

### Epic 2: Developer Experience Polish
- Better error messages
- Improved CLI ergonomics
- VS Code extension (syntax highlighting, run from editor)
- Configuration via simple YAML/TOML

### Epic 3: CI/CD Integration
- GitHub Actions example workflows
- GitLab CI templates
- Docker container with simulator pre-installed
- Headless mode for servers

### Epic 4: Community Building
- Public GitHub repo with clear contribution guide
- Discord/Slack community
- Issue tracker workflow
- Public roadmap

### Acceptance Criteria
- ✅ 100+ GitHub stars in first month
- ✅ 10+ external users running real firmware
- ✅ Documented examples for top 5 use cases
- ✅ Onboarding takes < 5 minutes

---

## 📦 Release v1.0 — "General Availability" (Months 13-14)

**Goal:** Production-ready release with first paying customers.

### Epic 1: Stability & Performance
- 1000+ hour fuzz testing
- Performance benchmarks (target: 50%+ realtime)
- Memory leak audits
- Crash reporting system

### Epic 2: Pricing & Licensing
- Open core split clearly defined
- License management for Pro features
- Stripe integration for subscriptions
- Customer dashboard

### Epic 3: Pro Features (Initial)
- Waveform recorder (capture all peripheral signals)
- Visual debugger UI (web-based)
- Code coverage reports for firmware
- Team collaboration features

### Epic 4: Enterprise Foundations
- Self-hosted deployment option
- SSO authentication
- Audit logging
- SLA documentation

### Acceptance Criteria
- ✅ First 10 paying customers
- ✅ Zero critical bugs in production
- ✅ Pricing pages live
- ✅ Pro features clearly differentiated

---

## 📦 Release v1.5 — "Multi-MCU" (Months 15-18)

**Goal:** Support multiple chip families to grow the addressable market.

### Epic 1: Additional ARM Cortex-M
- STM32F1 (Cortex-M3)
- STM32H7 (Cortex-M7)
- Nordic nRF52 (BLE-focused)

### Epic 2: Non-ARM Architectures
- **RP2040** (dual Cortex-M0+ — popular, legally clean)
- **ESP32** (Xtensa LX6 — huge IoT market)
- **RISC-V** target (ESP32-C3 or generic)

### Epic 3: Wireless Stack Modeling
- Wi-Fi behavioral simulation
- BLE peripheral simulation
- LoRa modem (UART command interface)

### Epic 4: Visual Tooling
- Web-based dashboard for live simulation
- Schematic-style view of connected devices
- Replay and time-travel debugging

### Acceptance Criteria
- ✅ Support 5+ MCU families
- ✅ Run ESP32 IoT firmware end-to-end
- ✅ 50+ paying customers

---

## 📦 Release v2.0 — "Enterprise & Cloud" (Months 19-24)

**Goal:** Scale to enterprise customers and cloud-based offerings.

### Epic 1: Cloud Platform
- SaaS offering (run simulations in browser)
- Team workspaces
- Shared test repositories
- Cloud-based CI runners

### Epic 2: Advanced Features
- Multi-board simulation (network of devices)
- Hardware-in-loop (HIL) bridge mode
- Real-time clock synchronization
- Power consumption modeling

### Epic 3: Enterprise Sales
- IP indemnification offering
- Custom MCU support contracts
- Training and certification program
- 24/7 support tier

### Epic 4: Ecosystem
- Plugin marketplace for custom devices
- Partner integrations (PlatformIO, Zephyr, etc.)
- Academic licensing program
- Industry-specific bundles (automotive, medical)

### Acceptance Criteria
- ✅ 5+ enterprise customers ($10K+ ARR each)
- ✅ Cloud platform handles 1000+ concurrent simulations
- ✅ Self-sustainable revenue

---

## 🔄 Sprint Cadence

| Activity | Frequency |
|---|---|
| Sprints | 2 weeks |
| Sprint planning | Every 2 weeks (2 hrs) |
| Standups | Daily (15 min) |
| Sprint review/demo | Every 2 weeks (1 hr) |
| Retrospective | Every 2 weeks (45 min) |
| Roadmap review | Monthly (2 hrs) |
| User interviews | Weekly (2-3 calls) |

---

## ⚠️ Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| CPU simulation accuracy issues | High | High | Use Unicorn Engine, validate against real hardware |
| Slow user adoption | Medium | High | Build in public, early user interviews |
| Competition copies features | Medium | Medium | Focus on developer experience as moat |
| Trademark/legal issues | Low | High | Careful naming, IP lawyer consultation |
| Performance not good enough | Medium | High | Profile early, optimize hot paths |
| Scope creep (too many MCUs) | High | Medium | Strict roadmap discipline, defer to v1.5+ |
| Team burnout (solo/small team) | Medium | High | Realistic timelines, milestone celebrations |

---

## 💰 Monetization Strategy

### Open Core Model

| Feature | Community (Free) | Pro ($99/mo) | Enterprise (Custom) |
|---|---|---|---|
| Core simulator | ✅ | ✅ | ✅ |
| Basic peripherals | ✅ | ✅ | ✅ |
| Common sensor library | ✅ | ✅ | ✅ |
| Pytest integration | ✅ | ✅ | ✅ |
| Waveform recorder | ❌ | ✅ | ✅ |
| Visual debugger | ❌ | ✅ | ✅ |
| Coverage reports | ❌ | ✅ | ✅ |
| Team workspaces | ❌ | ✅ | ✅ |
| Multiple MCU families | Limited | ✅ | ✅ |
| Self-hosted | ❌ | ❌ | ✅ |
| SSO / Audit logs | ❌ | ❌ | ✅ |
| Custom MCU support | ❌ | ❌ | ✅ |
| SLA & priority support | ❌ | ❌ | ✅ |
| IP indemnification | ❌ | ❌ | ✅ |

### Revenue Targets

| Period | MRR Target | Customer Count |
|---|---|---|
| End of Year 1 | $2K | 20 |
| End of Year 2 | $15K | 100 + 2 enterprise |
| End of Year 3 | $50K | 300 + 10 enterprise |

---

## ✅ Definition of Done (Per Feature)

A feature is **done** when:

- [ ] Code reviewed and merged
- [ ] Unit tests written and passing
- [ ] Integration test against real firmware
- [ ] Documentation updated
- [ ] Example added (if user-facing)
- [ ] Validated against real hardware (where applicable)
- [ ] Performance benchmarked (no regression)
- [ ] Released to community version (or gated for Pro)

---

## 📊 Key Performance Indicators

### Product Health
- Time-to-first-run (target: < 5 min)
- Simulation speed (target: > 50% realtime)
- Crash-free sessions (target: > 99%)

### Adoption
- Monthly active developers
- GitHub stars
- Community Discord members
- Documentation page views

### Business
- Free → Paid conversion rate (target: 5%)
- Monthly recurring revenue (MRR)
- Customer churn rate (target: < 3%/mo)
- Net Promoter Score (NPS, target: > 40)

---

## 🎓 Lessons from Existing Tools

| Tool | What to Learn | What to Avoid |
|---|---|---|
| **QEMU** | Powerful CPU emulation | Steep learning curve, poor DX |
| **Renode** | Multi-board scripting | Complex setup, niche community |
| **SimAVR** | Lightweight, focused | Limited to AVR |
| **Proteus** | Visual design appeals to some | Closed, expensive, outdated UI |
| **Wokwi** | Excellent web UX | Limited to specific scenarios |

**Our unique value:** Developer-first experience + modern testing workflows + open core flexibility.

---

## 🚀 Immediate Next Steps (Pre-Sprint)

1. **Validate market** — Interview 10+ embedded developers about testing pain
2. **Choose initial MCU** — Confirm STM32F4 vs alternatives based on user feedback
3. **Prototype CPU integration** — 2-week spike with Unicorn Engine
4. **Set up project infrastructure** — Repo, CI, issue tracker, communication
5. **Define brand & name** — Trademark search, domain registration
6. **Consult IP lawyer** — One-time review of licensing strategy
7. **Begin building in public** — Twitter/blog presence to attract early users

---

*Document version: 1.0 | Created: May 2026 | Owner: Project Lead*
