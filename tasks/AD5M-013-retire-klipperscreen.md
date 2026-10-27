# AD5M-013 Retire KlipperScreen

Status: complete

## Goal

Remove KlipperScreen and its X11/GTK touchscreen stack from the supported AD5M
build surface. The printer's approximately 128 MB RAM cannot support that
stack reliably enough to justify retaining it as an option.

## Context

ADR 0003 names HelixScreen as the only planned touchscreen UI. KlipperScreen
is an existing legacy build variant, not the selected future direction. Its
variant adds X11, Mesa, GTK, fonts, tslib, an init stack, a Git package, and
local patches; keeping it would preserve a resource-heavy and unused release
path.

## Repositories affected

- ad5m_enhanced_mod
- ad5m-project

## Requirements

- Remove the KlipperScreen Makefile target and all active build, package,
  overlay, post-build, image, and documentation references.
- Remove the now-unused tslib setup code when no supported variant calls it.
- Do not remove generated `build_output/` automatically.
- Keep historical task records accurate, but mark current instructions and
  architectural records as superseded where necessary.
- Validate the remaining `lite` configuration without building or installing
  a printer image.

## Non-goals

- Packaging or hardware-validating HelixScreen; that remains AD5M-005.
- Removing standard Buildroot components that are not selected by an active
  AD5M variant.
- Changing printer hardware, MCU firmware, motion, heaters, or safety limits.

## Verification

- `make -C ad5m_enhanced_mod help` lists only `lite`.
- `make -C ad5m_enhanced_mod -n klipperscreen` correctly fails because no such
  target exists.
- `bash -n ad5m_enhanced_mod/build_scripts/buildroot/post_build.sh` passed.
- `make -C ad5m_enhanced_mod lite_prepare` initialized the remaining external
  Buildroot configuration successfully. Buildroot emitted its existing merged
  configuration override warnings, but reported no error.
- A case-insensitive source scan found no KlipperScreen or Klipper Screen
  references in `ad5m_enhanced_mod` outside ignored generated output.
- Follow-up cleanup removed the unreferenced historical
  `printer_configs/KlipperScreen.conf`; it was not installed by the remaining
  `lite` variant and could otherwise suggest a supported configuration.

## Hardware verification

Not run. No image may be installed or MCU flashed for this removal task.
