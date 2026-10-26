# AD5M-003 Printer simulation and mock strategy

Status: proposed

## Goal

Define safe simulation and mocking layers for AD5M development, and document
their limits so they are not mistaken for real-printer validation.

## Context

There is no known full-fidelity emulator for the AD5M-specific SoC, MCU,
load-cell, heaters, motion system, display, and stock-host integration.
Useful partial substitutes exist: Klipper regression and batch processing,
mocked Klipper/Moonraker API responses, and generic QEMU userspace smoke
tests.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- Keep mocks and fixtures separate from production printer configuration.
- Test UI/API workflows using recorded or deliberately authored valid and
  failure-state Klipper/Moonraker responses.
- Use QEMU only for generic Buildroot userspace and service checks, not as a
  claim of AD5M hardware emulation.
- Define hardware-in-the-loop prerequisites and explicit safety boundaries for
  motion, heaters, probing, load-cell behavior, and MCU flashing.
- Record what each test layer can and cannot establish.

## Non-goals

- Emulating physical temperature, motion, load-cell, or safety behavior.
- Sending simulated/batch Klipper output to a real MCU.
- Replacing physical-printer testing.

## Proposed implementation

Create a short testing-strategy document and a minimal mocked API fixture for
one UI or workflow scenario. Add generic QEMU smoke coverage only after the
canonical Buildroot integration is established.

## Acceptance criteria

- The project records a test-layer matrix with explicit limits.
- At least one mock scenario can exercise a non-hardware workflow repeatably.
- No mock, QEMU, or batch result is labeled as hardware verification.

## Implementation notes

No implementation has started.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

Select the first workflow after the automated validation foundation exists.
