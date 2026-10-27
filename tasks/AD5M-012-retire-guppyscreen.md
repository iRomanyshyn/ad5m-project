# AD5M-012 Retire GuppyScreen

Status: complete

## Goal

Remove GuppyScreen from the supported AD5M build and release surface, and
record HelixScreen as the only planned touchscreen UI.

## Context

The project explicitly rejected GuppyScreen rather than completing its pending
updater and submodule-provenance reviews. ADR 0003 records the product and
security decision. HelixScreen remains separately scoped by AD5M-005; this task
does not claim that it is already packaged or validated.

## Repositories affected

- ad5m_enhanced_mod
- ad5m-project

## Implementation

- Removed the `guppyscreen` Makefile variant and its example command.
- Removed the Buildroot package registration, recipe, local patches, and
  variant configuration fragment, so its Git repository and submodules are no
  longer in the build graph.
- Removed the GuppyScreen overlay, init scripts, printer configuration, and
  post-build setup branch.
- Removed active GuppyScreen documentation and current developer instructions.
- Added ADR 0003 and marked AD5M-010 and AD5M-011 rejected, retaining their
  audit findings as historical records.

## Verification

- `make -C ad5m_enhanced_mod help` lists only `lite` and `klipperscreen`.
- `make -C ad5m_enhanced_mod -n guppyscreen` correctly fails because no such
  target exists.
- `bash -n ad5m_enhanced_mod/build_scripts/buildroot/post_build.sh` passed.
- `make -C ad5m_enhanced_mod lite_prepare` initialized the remaining external
  Buildroot configuration successfully. Buildroot emitted its existing merged
  configuration override warnings, but reported no error.
- A case-insensitive source scan found no GuppyScreen or Guppy references in
  `ad5m_enhanced_mod` outside ignored generated output.

### Follow-up regression check (2026-10-27)

AD5M-013 subsequently retired KlipperScreen as well, so the earlier
intermediate verification statement that listed it beside `lite` is historical
only.  A new static check confirms that `make -C ad5m_enhanced_mod help` lists
`lite` as the sole variant; dry-running both `guppyscreen` and
`klipperscreen` fails with "No rule to make target".  A case-insensitive scan
of active build scripts, overlays, Makefile, README and documentation found no
reference to either legacy screen.  `bash -n build_scripts/buildroot/post_build.sh`
also passed.  No Buildroot build or target image was started.

## Hardware verification

Not run. This task only removes an uninstalled build variant; it does not
modify or install a printer image, flash an MCU, or exercise printer hardware.
