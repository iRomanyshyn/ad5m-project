# AD5M-005 HelixScreen touch-layout ergonomics

Status: proposed

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

No implementation has started. Upstream documentation indicates Home-dashboard
widget editing is available; applicability to other panels remains to be
verified.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

Identify the current HelixScreen version, its installation/integration path,
and the specific controls that are problematic on the AD5M display.
