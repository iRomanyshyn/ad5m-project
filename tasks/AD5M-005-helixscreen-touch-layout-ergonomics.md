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

The official upstream repository's latest stable tag at the time of review is
[`v1.0.6`](https://github.com/prestonbrown/helixscreen/tree/v1.0.6), resolving
to immutable commit
[`11230009fa09b4509d0d05f482feaeef2e94c854`](https://github.com/prestonbrown/helixscreen/commit/11230009fa09b4509d0d05f482feaeef2e94c854).
It is GPL-3.0-or-later and is a C++17/LVGL application, not a Python/GTK
screen.  The annotated tag has no cryptographic signature.  It is the
candidate source for AD5M-015, not yet an accepted package source; the moving
`main` branch is not a release source.

Upstream has a Buildroot-oriented AD5M target,
`PLATFORM_TARGET=ad5m-br`, that deliberately uses the framebuffer backend,
the Buildroot cross-toolchain, dynamic target libraries, no SDL, no GLES, and
no screensaver.  In `v1.0.6` that target does *not* disable mock backends by
itself, so a production package must explicitly set `ENABLE_MOCKS=no` and
disable the remote-control server.  UI layout review still uses a native host
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

### 2026-10-27 configuration-boundary audit

The dashboard layout is genuinely upstream-configurable.  In `v1.0.6`, the
per-printer `panel_widgets/home` configuration stores up to eight pages and,
for every dashboard widget, its enabled state, grid column/row and column/row
span.  Entering Home edit mode disables the child controls while a user drags
or resizes widgets, then persists the result.  Given the project-owned
`HELIX_CONFIG_DIR` boundary, this makes whole-widget placement, empty grid
space, and separate pages viable without a HelixScreen fork.

There is also a concrete limitation.  The `control_buttons` widget places
Pause/Resume and Stop next to each other in one fixed horizontal row.  Its
internal layout has no corresponding dashboard setting, so moving or resizing
the widget cannot spatially separate those two actions.  Stop opens an
upstream confirmation dialog, which reduces the consequence of a mistap but
does not make the touch targets farther apart.  The printing E-Stop is a
separate floating top-right action with an upstream confirmation dialog; it
does not participate in the editable grid either.

The first AD5M mock prototype must therefore demonstrate two distinct cases:

1. a configuration-only Home layout that moves whole widgets and keeps a
   deliberate empty margin/page boundary around high-consequence controls;
   and
2. whether the adjacent Pause/Stop controls remain unacceptable on the actual
   800x480 resistive panel after their confirmation flow is reviewed.

If case 2 fails, record the measured touch problem and propose the smallest
upstreamable change: a split Pause/Resume and Stop widget, or a supported
widget option that selects a vertical/separated layout.  Do not patch a local
copy merely to change pixel coordinates.  A separate input setting can disable
Home edit mode or tune the global long-press delay, but that changes all
long-press gestures and is not a substitute for safer action placement.

### 2026-10-28 action-placement audit

The exact official `v1.0.6` root commit
`11230009fa09b4509d0d05f482feaeef2e94c854` was re-read in a temporary,
root-only checkout.  `git fsck --no-dangling` passed; no submodule was
initialised and no build, test, executable, installer or printer interaction
occurred.  The following is static source evidence for a future mock review,
not an assertion about the final AD5M screen.

| Surface | Placement control | Relevant safety behaviour | AD5M initial-layout implication |
| --- | --- | --- | --- |
| Home `Print Controls` widget | Whole widget can be put on a page, but its declared size is fixed at 2×1 cells; it has no resize range. Its XML is a horizontal flex row of equal Pause/Resume and Stop buttons. | Stop invokes the upstream cancel-confirmation modal. | Do not seed this widget on the main AD5M Home page. If it is offered later, put it on a deliberate controls page with surrounding empty cells; configuration cannot separate its two buttons. |
| Print Status panel | Not part of the Home grid. It has a fixed two-row action area: Pause shares a row with Light/optional Timelapse, while Cancel shares the next row with Tune. | Cancel uses the same confirmation flow. | Review this panel at native AD5M resolution; it cannot be made safe by Home dashboard placement. |
| Print Status E-Stop | Fixed 44×44 floating top-right button, outside the Home grid. | The safety manager defaults E-Stop confirmation to on and passes that setting to the action handler. | Preserve that default in the future first-run seed and test the confirmation target on the resistive display. |
| Advanced-panel emergency bar | Fixed panel layout; landscape places E-Stop, Restart Klipper and Firmware Restart in one row. Portrait alone stacks them. | E-Stop confirmation is governed by the same safety setting. | Treat it as a separate adjacent-control review on AD5M's landscape panel; Home configuration cannot improve it. |
| Home favourite macro | Configurable as a tile and can be moved/sized. | Upstream forces confirmation for macros classified dangerous (including `EMERGENCY_STOP` and `FIRMWARE_RESTART`); ordinary macro confirmation remains a safety setting. | Do not preseed high-consequence favourite macros until the AD5M macro mapping and confirmation flows are reviewed. |

This establishes an initial no-fork prototype policy: use the normal Home grid
for low-risk status/navigation widgets, leave deliberate empty space around a
controls-page widget if it is enabled, and rely on the dedicated Print Status
panel for active-print controls until physical testing demonstrates whether its
fixed layout is acceptable.  The package seed must be applied only on first
creation of the persistent configuration; it must never reset a user's edited
layout or safety choices on upgrade.

The highest-value upstream contribution, if physical testing finds the fixed
Home widget unacceptable, is a supported separated/vertical Print Controls
layout.  It should be proposed upstream with screenshots and touch evidence;
an AD5M-only pixel-coordinate fork remains out of scope.

## Verification

2026-10-28 static source audit only: the official `v1.0.6` root was checked
at `11230009fa09b4509d0d05f482feaeef2e94c854`; `git fsck --no-dangling`
passed.  No HelixScreen binary, mock mode, test suite, installer or package
build was run.

## Hardware verification

Not run.

## Remaining work

1. Complete AD5M-015's source and dependency closure before adding a package.
2. Build the pinned source natively and use `--test` plus a screenshot for the
   first dashboard prototype.
3. Build the no-fork candidate seed from the action-placement policy above and
   inventory any AD5M macro mappings before enabling favourite macro tiles.
4. Use the source-level boundary above to assess the fixed Home, Print Status,
   E-Stop and Advanced-panel controls in a mock screenshot review.
5. Create the AD5M-specific package, service, and persistent-config handoff
   only after the source review approves a fixed upstream version.
6. Perform a separately recorded physical touch and boot-transition review.
