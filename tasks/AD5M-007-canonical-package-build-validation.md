# AD5M-007 Canonical package-build validation

Status: proposed

## Goal

Build and inspect one or more Klipper Mod package variants using the canonical
Buildroot source contract established by AD5M-001, before considering any
printer installation.

## Context

AD5M-001 verified the external-tree configuration for SDK, lite,
KlipperScreen, and GuppyScreen variants, but deliberately did not download
package sources or compile firmware. A full build is required to validate the
source package set, toolchain, artifact creation, and disk-space assumptions.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- Start from the compatible commit pair recorded in AD5M-001, or explicitly
  record any reviewed successor pair.
- Confirm required disk space and network/source-download policy before
  beginning.
- Build the smallest useful variant first; do not assume all variants are
  equivalent.
- Record every external source downloaded, hash/commit pinning available, and
  any source or license issue discovered.
- Inspect package names, checksums, rootfs/chroot contents, and logs.
- Keep generated output local and out of Git.

## Non-goals

- Installing a package on a printer.
- Claiming motion, heater, MCU, display, or print validation.
- Updating package versions merely to make the build succeed.

## Proposed implementation

Measure available disk space, run the lite variant first using the canonical
contract, then inspect resulting output and build logs. Decide whether the UI
variants warrant separate builds based on their distinct dependencies.

## Acceptance criteria

- The selected variant completes or a precise reproducible failure is recorded.
- Generated artifacts can be identified and their integrity data inspected.
- The exact source commit pair and commands are recorded.
- No result is represented as a physical-printer test.

## Implementation notes

No implementation has started.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

Review the task's network, storage, and source-integrity prerequisites before
authorizing the build.
