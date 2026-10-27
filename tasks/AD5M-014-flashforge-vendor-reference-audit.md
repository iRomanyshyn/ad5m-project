# AD5M-014 Flashforge vendor-reference audit

Status: complete

## Goal

Establish a reproducible, security-reviewed way to consult Flashforge's
published AD5M Klipper source for hardware behavior and integration ideas.

## Context

The official Flashforge Adventurer 5M download page
(`https://www.flashforge.com/blogs/download-document/adventurer-5m`) lists
`AD5M_Series_Klipper` as firmware. The linked public repository identifies
itself as used on the Adventurer 5M series and carries GPL-3.0 text. This
baseline audit establishes a pinned review reference; it does not adopt code.

Community projects, including Z-mod and other AD5M mods, may also reveal an
idea or observable behavior. They are untrusted research inputs, not upstreams
or dependencies; see ADR 0004 and ADR 0001.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod, only if a later review approves a minimal implementation

## Requirements

- Independently verify that the vendor repository and the selected revision
  are official before relying on either; record its exact commit and license.
- Inventory only the files relevant to a defined question, such as AD5M
  hardware configuration, display integration, MCU behavior, or recovery.
- Compare a candidate behavior with the appropriate upstream project before
  considering a local change.
- Treat every external codebase, including vendor and community sources, as
  review input rather than an automatic code, binary, installer, script, or
  build dependency.
- For any candidate adoption, implement the smallest independent change and
  review privilege boundaries, downloads, updates, credentials, persistence,
  opaque assets, safety effects, and licensing.
- Record software and hardware verification separately in the follow-up task.

## Non-goals

- Cloning a vendor or community repository into the build graph.
- Merging, rebasing on, or bulk-importing another project's history.
- Assuming a backdoor or other defect without concrete evidence; the
  untrusted-input policy is a precaution, not an allegation.

## Acceptance criteria

- A reviewed source identity and exact revision are recorded for each question
  investigated.
- Every proposed adaptation has an explicit provenance and a scoped review.
- No unreviewed external code, binary, archive, submodule, updater, or
  download endpoint becomes part of an AD5M release.

## Audit record

On 2026-10-27, `main` and `HEAD` resolved to
`6f479ea5f23c1f69d569cc5d567e14707e97b7b2`. The official Flashforge download
page independently lists `AD5M_Series_Klipper` as an Adventurer 5M firmware
resource, providing vendor corroboration for the GitHub repository identity.

The commit carries an embedded OpenPGP signature using key ID
`B5690EEEBB952194`. Its cryptographic validity was not established: this
workstation has no trusted public key for it. It must not be treated as a
verified signer until a future task verifies the key's binding to Flashforge.

A temporary, shallow, `--filter=blob:none --no-checkout` clone was used only
for metadata and tree inspection outside the project and build graph. `git
fsck --no-dangling --connectivity-only` completed successfully. The pinned tree
has 1,917 paths and no `.gitmodules` file.

The top-level tree is a Klipper source layout (`config`, `docs`, `klippy`,
`lib`, `scripts`, `src`, and `test`) plus `fw-Ad5m`. The latter contains two
opaque firmware artifacts: `Eboard-20231012.hex` (92,484 bytes) and
`Mainboard-20230831.bin` (38,812 bytes). The tree also tracks `c_helper.so`
(180,576 bytes) and numerous Python `.pyc` files. None were executed,
installed, copied, or accepted as audit evidence beyond their presence and
metadata; none may be adopted without a separate, reproducible source and
safety review.

The filename-level inventory found no identified HelixScreen implementation or
AD5M-specific host display/UI source. That is not proof that no relevant
behavior exists in the tree; it means the next review must state a precise
question and selected source paths before reading implementation code.

## Outcome and follow-up

The vendor repository is suitable as a pinned, read-only reference for a
specific future question. It is not a dependency, source of firmware binaries,
or a replacement for official upstream projects. Any proposed reuse requires a
new scoped task that names the exact path and commit, compares the behavior
with upstream, and independently implements the minimal approved change.

HelixScreen integration remains AD5M-005. No current evidence in this audit
changes its scope or authorizes a HelixScreen package or printer deployment.

## Verification

- `git ls-remote` recorded the `main`/`HEAD` commit above.
- A temporary, no-checkout shallow clone was inspected outside the project;
  `git fsck --no-dangling --connectivity-only` passed.
- The source tree, submodule state, root layout, firmware/opaque-artifact
  paths, sizes, license heading, README, and commit signature metadata were
  inspected without executing source code or scripts.
- No project source, Buildroot configuration, package, firmware artifact, or
  printer state was changed by this audit.

## Hardware verification

Not run.
