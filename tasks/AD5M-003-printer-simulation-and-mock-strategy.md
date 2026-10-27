# AD5M-003 Printer simulation and mock strategy

Status: in progress

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

### 2026-10-27 emulator and test-harness audit

The canonical Buildroot checkout provides generic QEMU defconfigs, including
ARM `vexpress` and ARM64 `virt` machines, but none for the AD5M's Allwinner
T113-S3.  Its ARM vexpress example is a different board with a virtual
framebuffer and 256 MiB of RAM.  It can be useful later for generic userspace
and SysV-service smoke tests, but it cannot validate the AD5M boot chain,
kernel/device tree, framebuffer/touch wiring, camera, network hardware, MCU,
load-cell, heaters, or motion.  A QEMU result must therefore be labelled
`generic Buildroot userspace`, never `AD5M emulation`.

The tracked project and Klipper-Mod sources currently contain no project-owned
test runner, fixture directory, or CI workflow.  Local `build_output/` is not
a test harness and must stay ignored.  The first test foundation should be
small and source-controlled rather than adding an emulator image prematurely.

The proposed layered sequence is:

| Layer | First candidate | Establishes | Explicitly does not establish |
| --- | --- | --- | --- |
| Static | shell syntax, configuration policy and staged-file checks | package/config regressions | running services or hardware behaviour |
| Protocol fixture | authored Moonraker/Klipper JSON-RPC normal, disconnected and error states | UI/client request and failure handling | a live Moonraker, MCU or printer configuration |
| Host UI mock | HelixScreen `--test` screenshot after AD5M-015 source approval | visual layout and navigation | target binary, fbdev, touch, resource use or safety |
| Generic QEMU | a minimal separate ARM QEMU rootfs/service smoke test | generic BusyBox/SysV startup assumptions | any AD5M peripheral or boot claim |
| Hardware-in-the-loop | an explicitly prepared AD5M | final motion/thermal/touch/MCU evidence | unattended regression coverage |

The first fixture must be deliberately authored, versioned JSON rather than a
raw capture containing a printer name, network address, G-code, credentials,
or user data.  Start with the dashboard/control workflow: ready/idle,
printing, disconnected, and Moonraker error responses.  It must assert that a
mocked action is only requested, never relayed to a serial device or a real
Moonraker endpoint.  Adding that fixture is deferred until the consuming
client/interface and its exact API contract are accepted; at present no
project component can execute it.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

The first selected workflow is HelixScreen dashboard-layout review (AD5M-005).
The official HelixScreen `v1.0.6` candidate source, resolving to commit
`11230009fa09b4509d0d05f482feaeef2e94c854`, has a native `--test` mode
which uses a mock printer and can produce a screenshot.  Its AD5M Buildroot
target requires an explicit production `ENABLE_MOCKS=no` setting, so this is
strictly a host-side UI workflow, not a target-image or touch-driver test.

Before running it, AD5M-015 must select and audit the fixed upstream source
and its dependency closure.  The initial repeatable command sequence will be
recorded with the chosen source revision and will use an isolated
`HELIX_CONFIG_DIR`; it must not inherit a Moonraker host or any real-printer
settings.  The resulting screenshot establishes visual layout only.  It does
not validate resistive-touch accuracy, framebuffer startup, memory pressure,
or Klipper/Moonraker integration.

Generic QEMU work is a later, independent smoke-test layer.  It must use a
separate generic QEMU configuration and must not reuse or boot the AD5M image.
It is not a shortcut around physical validation.
