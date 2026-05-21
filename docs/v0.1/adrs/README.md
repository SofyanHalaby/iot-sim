# Architecture Decision Records — v0.1

ADRs capture the decisions we made and *why*, so a future contributor can understand the constraints we were working under. We use a lightweight MADR-inspired template.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-language-and-standard.md) | Language and standard: C++20 | Accepted |
| [0002](0002-cpu-backend-and-license-isolation.md) | CPU backend: Unicorn 2.x in a subprocess for GPL isolation | Accepted |
| [0003](0003-build-system.md) | Build system: CMake + CMakePresets + vcpkg | Accepted |
| [0004](0004-test-framework.md) | Test framework: GoogleTest + GoogleMock | Accepted |
| [0005](0005-sanitizers-and-ci.md) | Sanitizers and CI matrix | Accepted |
| [0006](0006-initial-mcu-target.md) | Initial MCU target: STM32F407VG / STM32F4-Discovery | Accepted |

## Template

```markdown
# ADR NNNN: <Title>

**Status:** Proposed | Accepted | Superseded by [#NNNN](NNNN-...)
**Date:** YYYY-MM-DD

## Context
What forces are at play? What constraints, requirements, or prior decisions led
to this one needing to be made? Keep this honest — context decays fast.

## Decision
The decision, stated as a single declarative sentence followed by any specifics.

## Consequences
What becomes easier as a result. What becomes harder. What new obligations does
this place on contributors. What we'll have to revisit later.

## Alternatives considered
Other options we looked at, and the specific reason each was rejected. Be
charitable to the rejected options — they may come back.
```

## Conventions

- File name: `NNNN-kebab-case-title.md`. NNNN is a monotonic, zero-padded counter scoped per release folder.
- Once **Accepted**, the body is immutable except for status changes. A new ADR supersedes; the old one is updated to point at the new one.
- Cross-link generously. `[ADR 0002](0002-...)` style.
- An ADR exists when the decision would be expensive to reverse or surprising to a newcomer. Not every choice needs one.
