# AD5M-001 Buildroot integration contract

Status: proposed
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

No implementation has started.

## Verification

Not run.

## Hardware verification

Not applicable until a package-changing implementation is proposed.

## Remaining work

Inspect the build interface and choose the minimum safe migration approach.
