# AD5M-005 HelixScreen touch-layout ergonomics

Status: in progress

## Goal

Create and validate a safer, more ergonomic HelixScreen layout for the AD5M's
resistive touchscreen by separating controls that are easy to mistap.

## Context

HelixScreen documentation supports a configurable multi-page Home dashboard:
widgets can be enabled, moved, and resized in Edit Mode. This may be sufficient
to separate high-consequence controls without modifying HelixScreen itself.
The documentation does not establish that every built-in control panel has an
equally flexible user-configurable layout.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Start with upstream-supported configuration and widget placement; do not fork
  or patch HelixScreen unless a concrete limitation is demonstrated.
- Identify control pairs that are too close together and classify their
  consequence if mistapped.
- Keep high-consequence actions visually and spatially distinct; use upstream
  confirmation mechanisms where they are available.
- Evaluate layout changes in HelixScreen's mock/test mode and capture
  screenshots before printer deployment.
- Test touch behavior on the physical resistive display before describing the
  layout as validated.
- Keep all user-customized HelixScreen settings persistent across upgrades.

## Non-goals

- Weakening heater, motion, power, or emergency-stop safety behavior.
- Claiming a desktop/mock screenshot validates resistive-touch usability.
- Maintaining a permanent HelixScreen fork without an upstream-worthy reason.

## Proposed implementation

Inventory the AD5M dashboard and adjacent controls, prototype widget placement
and sizing with HelixScreen configuration, and test it using the upstream mock
mode and screenshots. If an important panel cannot be made safe through
configuration, document the exact limitation before considering a small
upstreamable HelixScreen change.

## Acceptance criteria

- The proposed layout and the rationale for each moved or resized control are
  documented.
- A mock-mode screenshot review is recorded.
- Physical touch testing is explicitly recorded separately.
- Any HelixScreen source change is justified and suitable for upstream review.

## Implementation notes

### 2026-10-27 upstream discovery

The official upstream repository was examined at immutable commit
[`6eb61edc55244d62cc76a294203bbb9e012161ef`](https://github.com/prestonbrown/helixscreen/commit/6eb61edc55244d62cc76a294203bbb9e012161ef).
It is GPL-3.0-or-later and is a C++17/LVGL application, not a Python/GTK
screen.  Its tracked `VERSION.txt` identifies that snapshot as
`1.1.0-beta.13`, despite the repository README describing a stable 1.0
release.  We must choose a reviewed release/tag or a specifically reviewed
commit before packaging; the moving `main` branch is not a release source.

Upstream has a Buildroot-oriented AD5M target,
`PLATFORM_TARGET=ad5m-br`, that deliberately uses the framebuffer backend,
the Buildroot cross-toolchain, dynamic target libraries, and no SDL.  It
explicitly disables its mock backend.  The product image must therefore be a
separate Buildroot package, while UI layout review must use a native host
build with `--test`; no mock result represents an AD5M runtime image or its
touch panel.

The source supports `HELIX_CONFIG_DIR` as a writable configuration directory
independent of its read-only assets.  The intended integration will keep
program assets below `/opt/helixscreen` and put persistent user settings,
including dashboard layout, below
`/root/printer_data/config/helixscreen`.  This path already belongs to the
project's persistent Klipper/Moonraker data area.  The future init service
must set the directory itself, rather than accept it from a user-editable
environment file.

Upstream's Home dashboard supports an editable, multi-page grid and macro
confirmation.  The next prototype will use that upstream configuration only:
put high-consequence controls on a separate page or with an empty grid margin,
require confirmation where the upstream control supports it, and retain an
explicit record of controls that cannot be rearranged.

The generic `lite` image still contains the static `S90welcome` framebuffer
image.  It is not a KlipperScreen service and remains required for `lite`.
The HelixScreen-specific init service must own the display after boot and its
ordering/transition must be reviewed as part of the packaging task.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

1. Complete AD5M-015's source and dependency closure before adding a package.
2. Build the pinned source natively and use `--test` plus a screenshot for the
   first dashboard prototype.
3. Inventory the AD5M controls that are adjacent today; record their
   consequence and the specific grid/confirmation treatment.
4. Create the AD5M-specific package, service, and persistent-config handoff
   only after the source review approves a fixed upstream version.
5. Perform a separately recorded physical touch and boot-transition review.
