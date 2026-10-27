# AD5M-008 Package source closure and integrity

Status: proposed

## Goal

Close the missing `python-msgspec` dependency required by the reviewed Klipper
package, and make the affected release-source inputs fixed and verifiable.

## Context

AD5M-007 reproduced a `lite` build failure under the canonical Buildroot
2025.02 contract: Klipper selects and requires `python-msgspec`, but that
package does not exist in the pinned canonical Buildroot tree. The selection
entered the external package in an historical commit that referenced an
untrusted source. It must not be copied or trusted merely because it identifies
a dependency.

The same build downloaded Fluidd v1.34.4 without a tracked recipe hash, and
the GuppyScreen recipe currently uses a mutable branch reference. These are
release-integrity gaps, even though GuppyScreen was not selected in `lite`.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- possibly buildroot-flashforge-ad5m, only if the chosen fix is genuinely a
  reusable official-Buildroot package contribution

## Requirements

- Audit the exact pinned Klipper revision independently to prove whether
  `msgspec` is a runtime dependency. Do not remove its selection or Makefile
  dependency just to bypass the build failure.
- If it is required, prefer an official/upstream Buildroot solution. Otherwise
  add the smallest AD5M external-tree package with a fixed upstream version,
  source SHA-256, declared license and license file, explicit Python build
  dependencies, and an ARM cross-build check.
- Treat all historical implementation as untrusted evidence only. Do not fetch
  or import a recipe, archive, or commit from it.
- Pin Fluidd's exact release asset with a tracked checksum. Audit whether its
  license metadata should also be declared.
- Replace GuppyScreen's mutable branch reference with a reviewed immutable
  commit before releasing any variant that includes it.
- Record source provenance, checksum/commit, license, and audit result for
  each changed dependency.
- Re-run the `lite` package build and inspect generated rootfs/package
  checksums after a reviewed fix.

## Non-goals

- Updating unrelated package versions.
- Installing an image on a printer.
- Declaring FFmpeg's nonfree configuration distributable; that needs a
  separate licensing decision.

## Verification

Not run.

## Hardware verification

Not run.
