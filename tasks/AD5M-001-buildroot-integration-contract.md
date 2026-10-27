# AD5M-001 Buildroot integration contract

Status: complete
Priority: first implementation task

## Goal

Replace the Klipper Mod build system's legacy historical Buildroot submodule
dependency with a reviewed, explicit relationship to the canonical
`buildroot-flashforge-ad5m` repository.

## Context

The current Klipper Mod Makefile expects `submodules/buildroot`, whose URL is a
historical third-party repository. The canonical base repository instead tracks
official Buildroot on `ad5m-2025.02`. This prevents reproducing a package from
the three canonical repositories alone and violates the project's untrusted
historical-source policy.

This is intentionally the first implementation task. Firmware features should
not be built on top of a Buildroot source relationship that the project intends
to remove.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- Do not initialize, fetch, merge, or import the historical submodule as part
  of the solution.
- Establish and document the exact compatible commit pair or other reviewed
  version contract between the two canonical source repositories.
- Preserve official Buildroot as the authoritative upstream.
- Keep AD5M-specific external-tree content in `ad5m_enhanced_mod` and generic
  Buildroot content in `buildroot-flashforge-ad5m`.
- Ensure generated output remains local and ignored.
- Do not treat this as a URL-only submodule replacement. Audit the exact delta
  between the Buildroot revision expected by the current Klipper Mod build
  system and the canonical official-Buildroot-derived repository.
- Identify which required behavior already lives in the Klipper Mod external
  tree, which is present in official Buildroot 2025.02, and which genuinely
  requires a change to the canonical Buildroot repository.
- Prefer the smallest explicit integration contract that keeps the two source
  repositories independently maintainable.

## Non-goals

- Adding firmware features or changing printer behavior.
- Updating arbitrary Buildroot packages.
- Creating a release package before the build contract is verified.

## Proposed implementation

Inspect the existing build wrapper and external-tree interface. Before changing
`.gitmodules` or source-selection logic, compare the legacy Buildroot revision
used by the current build with the canonical `ad5m-2025.02` tree and classify
every required difference.

Then define a reviewed source-selection mechanism that uses the canonical
Buildroot checkout or a clearly pinned clone of it, remove the legacy submodule
requirement, and document the bootstrap/build commands in DEVELOPMENT.md.

A simple replacement of the legacy submodule URL is not sufficient unless the
audit proves the canonical tree already provides every required dependency and
behavior.

## Acceptance criteria

- A fresh workstation can obtain every required source from the three
  canonical repositories and their documented official upstreams.
- No routine build path references `loss-and-quick` or the legacy submodule.
- The selected source revisions and verification command are documented.
- Software-level validation is run and recorded; no hardware claim is made
  without a physical-printer test.

## Implementation notes

- The historical gitlink was pinned to `ae9518dc7d2db7bad68a66020c06593554f970ee`,
  but its object was not present locally and was not fetched or initialized.
- The canonical Buildroot source at
  `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de` is two documentation commits
  after official tag `2025.02.19`; its only delta from that tag is `AGENTS.md`.
- The AD5M external tree and `sdk + base + lite` fragments were accepted by
  canonical Buildroot `olddefconfig`. The base fragment needed an explicit
  `BR2_PACKAGE_LIBGPIOD=y` parent so that the already requested tools option
  is retained.
- A second historical source reference was found in the backlight package.
  The exact pinned source commit was fetched only from
  `consp/flashforge_ad5m_backlight` for review, not from the historical source.
  Its tracked implementation uses `/dev/disp` ioctls and contains no network
  client code; its unused release helper can upload to PyPI.
- The build wrapper's pre-existing `git describe` command failed because this
  fork has no tag. It now has a tag-or-SHA fallback.
- The wrapper now uses and validates the canonical Buildroot contract; the
  legacy gitlink and `.gitmodules` entry were removed.
- Compatible source revisions validated together:
  - Klipper Mod: `b33f8cc9d8c1955a542caccaec31983d786d6da3`
  - Buildroot: `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de`

## Verification

- `bash -n build_scripts/env.sh`
- `bash -n build_scripts/build.sh`
- `make -C ad5m_enhanced_mod help`
- `make -C ad5m_enhanced_mod sdk_prepare`
- `make -C ad5m_enhanced_mod lite_prepare`
- `make -C ad5m_enhanced_mod klipperscreen_prepare`
- `make -C ad5m_enhanced_mod guppyscreen_prepare`
- `AD5M_BUILDROOT_DIR=/tmp/not-a-buildroot-tree make -C ad5m_enhanced_mod sdk_prepare`
  (expected rejection)

All four configuration-only prepare targets completed against the canonical
Buildroot source. No Buildroot package target or firmware build was run.

## Hardware verification

Not run. This work changes build integration only; a future package build and
printer installation still require separate hardware verification.

## Remaining work

Run a full package build and inspect artifacts in
`tasks/AD5M-007-canonical-package-build-validation.md` before any printer
installation.
