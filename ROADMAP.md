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

## Reliability

- pause and filament-runout timeout behavior
- MCU timeout investigation
- G-code transfer/upload integrity
- persistent user configuration
- safer installer/update mechanism
- rollback/recovery
- MCU update recovery

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
- diagnostics export
- establish a layered test strategy: static checks, Klipper regression and
  batch G-code tests, mocked Klipper/Moonraker API scenarios, generic QEMU
  userspace smoke tests, and hardware-in-the-loop testing; see
  `tasks/AD5M-002-automated-validation-foundation.md` and
  `tasks/AD5M-003-printer-simulation-and-mock-strategy.md`
