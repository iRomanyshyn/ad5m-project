# AD5M Project Roadmap

This file is a high-level backlog.

Actual implementation work should use scoped task files under tasks/.

A task with `Status: proposed` records an idea or candidate work item; it is
not authorization to implement it and does not by itself define execution
order. Review proposed tasks with the user before starting them.

The current first implementation priority is
`tasks/AD5M-001-buildroot-integration-contract.md`. Establish the canonical
Buildroot relationship before firmware features are developed on top of the
legacy submodule dependency.

## Project foundations and reproducibility

- replace the legacy historical Buildroot submodule with a reviewed build
  integration contract; see `tasks/AD5M-001-buildroot-integration-contract.md`
- run a full package build from the canonical source contract and inspect its
  artifacts before any printer installation; see
  `tasks/AD5M-007-canonical-package-build-validation.md`
- resolve the reviewed Klipper `python-msgspec` dependency closure and pin
  mutable/unhashed release sources before retrying the package build; see
  `tasks/AD5M-008-package-source-closure-and-integrity.md`
- choose a legally reviewable FFmpeg distribution profile before publishing an
  image; see `tasks/AD5M-009-ffmpeg-distribution-profile.md`
- record a compatibility matrix for both printer variants, board/MCU revisions,
  stock-host versions, and validated cross-repository commit pairs
- define a release manifest with component revisions and hashes; add SBOM,
  license review, release integrity, and signing as appropriate
- define recovery and fault-injection expectations for power loss, storage
  exhaustion, interrupted updates, MCU loss, and network loss
- define a hardware test matrix that keeps software validation distinct from
  physical-printer validation
- in progress: close the non-hardware personal-alpha readiness plan before
  any target installation; see
  `tasks/AD5M-031-pre-hardware-readiness-and-personal-alpha-plan.md`

## Reliability

- pause and filament-runout timeout behavior
- MCU timeout investigation
- G-code transfer/upload integrity
- persistent user configuration
- safer installer/update mechanism
- proposed: make the legacy installer reject invalid archives and inadequate
  storage before it deletes an existing chroot; see
  `tasks/AD5M-027-installer-preflight-before-destruction.md`
- rollback/recovery
- MCU update recovery
- proposed: make legacy MCU update success/failure accounting explicit before
  relying on any recovery marker; see
  `tasks/AD5M-034-mcu-update-result-accounting.md`
- proposed: establish a board-revision-specific recovery and reflashing
  capability before the first project firmware installation; see
  `tasks/AD5M-026-recovery-and-reflashing-readiness.md`

## Safety

- review heater verification configuration
- review temperature limits
- probing/load-cell safety
- MCU flashing failure handling
- shutdown/power behavior

## Bed leveling and nozzle preparation

- native adaptive bed mesh
- remove unnecessary thermal cycling during mesh preparation
- improve nozzle cleaning before probing
- optional physical nozzle brush/wiper
- retain explicit full-bed calibration fallback

## UI

- HelixScreen integration; it is the only planned touchscreen UI (see
  `decisions/0003-screen-platform-policy.md`)
- proposed: establish a pinned, first-class Buildroot package and persistent
  configuration boundary before enabling HelixScreen in an image; see
  `tasks/AD5M-015-helixscreen-source-and-buildroot-integration.md`
- provide a documented primary web UI without requiring users to enter
  non-standard port `4001`; implementation remains blocked on the reviewed
  access policy; see
  `tasks/AD5M-004-stable-web-and-camera-access.md`
- keep camera configuration stable when the printer address changes; prefer a
  reviewed stable name or same-origin/proxy design over a raw IP address; see
  `tasks/AD5M-004-stable-web-and-camera-access.md`
- proposed: establish a per-printer persistent mDNS identity before documenting
  stable UI, camera or slicer addresses; see
  `tasks/AD5M-017-persistent-device-name-and-mdns.md`
- proposed: evaluate an ergonomic HelixScreen layout that separates
  easy-to-mistap controls on the resistive display; review the task before
  implementation; see
  `tasks/AD5M-005-helixscreen-touch-layout-ergonomics.md`
- Wi-Fi/network configuration
- filament load workflow
- filament unload workflow
- automatic preheat for extrude/retract
- purge speed units/defaults
- stale filament requests/prompts
- diagnostics/status display

## Buildroot and system

- official Buildroot 2025.02 LTS maintenance
- audit historical AD5M Buildroot modifications
- dependency completeness
- memory/resource management; measure real pressure before changing defaults;
  see `tasks/AD5M-006-memory-pressure-zram-swap.md`
- logging limits
- evaluate zram first, then internal-flash or USB-backed swap only if justified
  by measurements and failure-mode testing; see
  `tasks/AD5M-006-memory-pressure-zram-swap.md`
- establish the actual display, camera and USB topology before using a board
  header or internal storage; see
  `tasks/AD5M-024-ad5m-display-camera-usb-topology.md`
- service startup optimization

## Security

- in progress: produce a source-only provenance and privileged-execution
  findings report before accepting further release-facing features; see
  `tasks/AD5M-028-phase-zero-provenance-and-privileged-path-audit.md`
- proposed: replace the shared root password with a per-printer, recoverable
  bootstrap flow before release; see
  `tasks/AD5M-029-credential-bootstrap-and-ssh-hardening.md`
- proposed: define all system egress, including NTP, independently of the
  HelixScreen local-only profile; see
  `tasks/AD5M-032-system-network-egress-policy.md`
- remove automatic privileged execution of code supplied on removable media
  and define an explicit, verifiable recovery-media trust boundary; see
  `tasks/AD5M-025-removable-media-host-trust-boundary.md`
- proposed: define Moonraker authorization, trusted-client and CORS policy
  before changing browser endpoints; see
  `tasks/AD5M-016-moonraker-lan-authorization-and-browser-origins.md`
- SSH credential handling
- firewall/network exposure
- release integrity
- SHA-256/signing

## Tooling

- CI
- automated configuration validation
- shell/Python linting
- reproducible build documentation
- component/version manifest
- proposed: audit the pinned Klipper fork and all local safety-affecting
  patches before treating its generated MCU firmware as releaseable; see
  `tasks/AD5M-030-klipper-source-and-safety-patch-audit.md`
- proposed: account for or eliminate tracked binary kernel modules without
  source provenance; see `tasks/AD5M-033-binary-kernel-module-provenance.md`
- diagnostics export
- establish a layered test strategy: static checks, Klipper regression and
  batch G-code tests, mocked Klipper/Moonraker API scenarios, generic QEMU
  userspace smoke tests, and hardware-in-the-loop testing; see
  `tasks/AD5M-002-automated-validation-foundation.md` and
  `tasks/AD5M-003-printer-simulation-and-mock-strategy.md`
