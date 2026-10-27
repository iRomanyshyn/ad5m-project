# AD5M-014 Flashforge vendor-reference audit

Status: proposed

## Goal

Establish a reproducible, security-reviewed way to consult Flashforge's
published AD5M Klipper source for hardware behavior and integration ideas.

## Context

Initial discovery found the public repository
`https://github.com/FlashForge/AD5M_Series_Klipper`, which identifies itself
as used on the Adventurer 5M series and is GPL-3.0 licensed. The repository
contains printer-specific configuration, `klippy`, firmware, scripts, and
documentation. This discovery does not establish a pinned revision or adopt
any code.

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

## Verification

Not run. Initial web discovery only; no repository was cloned, fetched into
the build, or used as an implementation source.

## Hardware verification

Not run.
