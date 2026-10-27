# AD5M-015 HelixScreen source and Buildroot integration

Status: proposed

## Goal

Create a first-class, reproducible HelixScreen package and service for the
AD5M external Buildroot tree, with a persistent configuration boundary and no
dependence on third-party AD5M installers or firmware mods.

## Context

HelixScreen is the project's sole planned touchscreen UI under ADR 0003.
The official upstream repository provides an AD5M-specific Buildroot target
(`PLATFORM_TARGET=ad5m-br`) and an install staging target, but its documented
AD5M installer is an imperative script for other firmware variants.  It must
not be run or imported into this project.

An upstream snapshot was examined on 2026-10-27 at commit
[`6eb61edc55244d62cc76a294203bbb9e012161ef`](https://github.com/prestonbrown/helixscreen/commit/6eb61edc55244d62cc76a294203bbb9e012161ef).
It is GPL-3.0-or-later, carries a `PLATFORM_TARGET=ad5m-br` path, installs
assets under `/opt/helixscreen`, and supports a separate writable
`HELIX_CONFIG_DIR`.  Its `VERSION.txt` says `1.1.0-beta.13`; that differs from
the repository README's stable-release statement, so the package version must
be selected explicitly rather than following `main`.

The source has 14 Git submodules at this snapshot: cpp-terminal, glm,
helix-xml, libhv, libjpeg-turbo, libnl, linenoise, Lua, lv_i18n,
lv_markdown, LVGL, SDL2, spdlog, and a WPA-supplicant client.  Their Gitlink
commits are immutable only as references; each source and license still needs
closure review before a recursive-source package may be enabled.

## Repositories affected

- ad5m_enhanced_mod
- ad5m-project

## Requirements

- Pin an official upstream release/tag or immutable commit and record the
  source identity, archive/Git integrity evidence, license, and release
  selection rationale.
- Review every fetched submodule and external dependency for source identity,
  license, Buildroot ownership, and whether it can be replaced by an existing
  Buildroot package.  Do not enable recursive Git checkout before that review.
- Keep the package in `ad5m_enhanced_mod` unless a dependency is demonstrably
  generic Buildroot functionality.
- Build the upstream `ad5m-br` target through the canonical Buildroot SDK;
  do not use upstream Docker toolchains or deployment targets.
- Use direct `fbdev` display output; do not add X11, Wayland, GTK, or a desktop
  stack.
- Install immutable application assets under `/opt/helixscreen`; persist
  `settings.json`, `helixscreen.env`, themes, and user layout under
  `/root/printer_data/config/helixscreen` through a package-owned service.
- Do not execute, ship, source, or patch in any upstream installer/deployer.
  Do not rely on Forge-X, ZMOD, Klipper Mod, or other community integration
  scripts.
- Provide an AD5M-specific SysV service that starts after the needed printer
  services without blocking their startup, handles a failed UI gracefully, and
  does not stop Klipper or Moonraker.
- Replace the `lite` welcome framebuffer only in the HelixScreen variant;
  retain `lite` as a minimal no-GUI image.
- Add software-only checks for package selection, service syntax, staged file
  ownership, and the absence of legacy-screen packages.  Keep physical display,
  touch, boot, and memory validation as separate acceptance evidence.

## Non-goals

- Running an installer, update script, or deploy command on any printer.
- Changing Klipper safety configuration, MCU firmware, heaters, motion,
  probing, or power control.
- Treating a host mock screenshot as target or hardware validation.
- Adding in-application self-update before a transactional project update
  design exists.

## Proposed implementation

1. Resolve the selected official source and complete source/dependency closure
   using AD5M-008's integrity standard.
2. Add a project-specific Buildroot package that invokes only the upstream
   `ad5m-br` build and staging interfaces, with an explicit, reviewed set of
   target library dependencies.
3. Add a separate `helixscreen` variant instead of changing `lite` in place.
4. Add a small project-owned SysV init script and fixed environment that sets
   `HELIX_DISPLAY_BACKEND=fbdev`, the persistent `HELIX_CONFIG_DIR`, and only
   hardware values verified on an AD5M.
5. Add an upgrade-safe seed/migration policy for default settings without
   overwriting user layout or connection settings.
6. Validate build/configuration and host mock separately; schedule physical
   testing only after an installable image exists.

## Acceptance criteria

- A reviewed immutable source and all recursively fetched source identities
  are recorded, including licenses and integrity checks.
- `make helixscreen_prepare` configures a separate no-legacy-screen variant.
- The target package builds from the canonical pinned Buildroot source without
  upstream Docker images or installers.
- The staged image has `/opt/helixscreen` assets and a writable persistent
  configuration directory, with no default setting that overwrites a user's
  dashboard layout on rebuild/upgrade.
- The service does not retain a competing UI and fails without destabilizing
  Klipper or Moonraker.
- Host mock, target build, and physical-printer evidence are reported as three
  distinct test layers.

## Verification

Upstream source inspection only.  No package has been created, no source
submodule has been accepted, and no HelixScreen binary, installer, updater,
or deployment script has been run.

## Hardware verification

Not run.

## Remaining work

- Select a stable, reviewable official upstream version.
- Audit the complete source/dependency graph and target libraries.
- Measure the packaged image's disk/RAM impact and boot behavior.
- Determine exact framebuffer, touch device, calibration, and backlight values
  from a physical AD5M before setting hardware-specific environment values.
