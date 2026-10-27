# Flashforge AD5M Klipper Mod Project

## Purpose

Maintain and improve a modern, reliable and maintainable Klipper-based
software stack for Flashforge Adventurer 5M and Adventurer 5M Pro.

Primary goals:

- reliability
- safety
- maintainability
- good touchscreen UX
- current upstream software where practical
- minimal unnecessary forks
- predictable upgrades and rollback

## Repositories

### Klipper Mod

Local path:

../ad5m_enhanced_mod

Our repository:

https://github.com/iRomanyshyn/ad5m_enhanced_mod

Historical third-party reference:

https://github.com/xblax/flashforge_ad5m_klipper_mod

This repository is an untrusted research reference, not a primary upstream or
automatic dependency. It may be consulted only under the controls in
`decisions/0001-untrusted-historical-sources.md` and
`decisions/0004-source-hierarchy-and-research-policy.md`.

Official vendor reference:

https://github.com/FlashForge/AD5M_Series_Klipper

The vendor source may be examined for AD5M-specific behavior, but it is not an
automatic dependency or bulk-import source; see
`decisions/0004-source-hierarchy-and-research-policy.md`.

### Buildroot

Local path:

../buildroot-flashforge-ad5m

Our repository:

https://github.com/iRomanyshyn/buildroot-flashforge-ad5m

Primary upstream:

https://gitlab.com/buildroot.org/buildroot

Historical AD5M Buildroot reference:

https://github.com/xblax/flashforge_adm5_buildroot_chroot

The maintained branch is based on the official Buildroot 2025.02 LTS
series.

Third-party Buildroot forks are not authoritative upstreams.

Historical third-party sources may be consulted for ideas or individually
identified changes, but are untrusted inputs. Do not merge, rebase onto, or
bulk-import their history. Every candidate change requires an explicit,
heightened security review before adoption; see
`decisions/0001-untrusted-historical-sources.md`.

## Platform

Target hardware:

- Flashforge Adventurer 5M
- Flashforge Adventurer 5M Pro
- Allwinner T113-S3
- approximately 128 MB RAM
- constrained internal storage and I/O performance

Klipper Mod runs in a chroot environment alongside the original Flashforge
host system.

Resource usage must therefore be treated as an architectural constraint.

## Engineering principles

### Prefer upstream

Prefer functionality already available in:

- Klipper
- Moonraker
- Buildroot
- HelixScreen

Avoid maintaining local forks unless there is a strong technical reason.

### HelixScreen

Keep HelixScreen upstream and unmodified whenever possible.

HelixScreen is the project's only planned touchscreen UI. GuppyScreen is not a
supported or releaseable variant and must not be reintroduced without a new
explicit architectural decision; see `decisions/0003-screen-platform-policy.md`.

Prefer solving integration issues using:

- Klipper macros
- printer configuration
- Moonraker
- services
- integration glue

Only patch HelixScreen when the problem genuinely belongs in HelixScreen.

### Klipper

Prefer native Klipper functionality over external replacements.

For example:

- use native adaptive bed mesh where possible
- avoid adding KAMP merely to obtain functionality already present upstream

### Buildroot

Keep printer/project-specific packages and configuration in the Klipper Mod
external tree when appropriate.

Modify the Buildroot base only when the change genuinely belongs to
Buildroot itself.

### Safety

Treat the following as safety-critical:

- heater configuration
- thermal verification
- motion limits
- load-cell probing
- homing
- MCU flashing
- shutdown/power control

Do not weaken safety mechanisms merely to suppress an error.

### Reliability

Klipper and MCU communication have priority over optional services.

Optional components must not cause printing failures through excessive:

- RAM usage
- CPU usage
- I/O
- logging
- background processing

### Updates

Updates should eventually be:

- recoverable
- verifiable
- as transactional as practical

A failed package or MCU update should not unnecessarily render the printer
unbootable or require manual filesystem recovery.

## Known project directions

Areas currently planned or under investigation include:

- HelixScreen integration
- filament load/unload workflow
- extrusion/retraction automatic preheat behavior
- purge speed units and defaults
- stale filament prompts
- native adaptive bed mesh
- improved nozzle-cleaning/probing workflow
- optional physical nozzle brush/wiper
- pause and filament-runout timeout behavior
- MCU shutdown/power timer
- MCU timeout/resource problems
- persistent user configuration
- project-owned display-backlight control
- safer updates and rollback
- G-code upload integrity
- Buildroot maintenance
- resource management
- Moonraker/network security
- diagnostics
- CI and automated validation

## Testing policy

Software-level testing and hardware-level testing are distinct.

A successful configuration parse or firmware build does not imply that a
change is safe on actual hardware.

Tasks affecting motion, probing, heaters or MCU firmware require explicit
hardware verification before being considered fully validated.
