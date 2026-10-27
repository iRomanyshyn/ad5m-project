# AD5M-010 GuppyScreen update-path audit

Status: rejected

## Decision

Rejected on 2026-10-27. AD5M-012 retired GuppyScreen and ADR 0003 makes
HelixScreen the only planned touchscreen UI. This audit is not to be resumed
or used to make a GuppyScreen release possible. Its analysis remains as a
historical record of why an unreviewed updater must not enter a firmware image.

## Former goal

Decide whether GuppyScreen's visible update control is disabled, replaced with
the project's reviewed update flow, or supplied with a fully verified updater
before releasing the GuppyScreen variant.

## Context

AD5M-008 audited GuppyScreen commit
`fdca5728001f7a6f6ce774dcfa3998f8b96e6878`. Its Settings panel resolves and
executes an adjacent `update.sh` when the update control is pressed. The
upstream script downloads a replacement `curl` with `wget --no-check-certificate`,
queries a mutable GitHub releases list, downloads an unsigned/unhashed archive,
extracts it over the installed program, and can run an adjacent custom upgrade
script.

The present Buildroot recipe installs only the reviewed `guppyscreen` binary,
not `update.sh`; the button therefore finds no script in the packaged image.
This avoids the unsafe update path today, but leaves a confusing UI action and
must not be relied on as an implicit long-term security boundary.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Historical audit requirements

- Keep the upstream `update.sh`, installers, and release archives out of the
  firmware image unless a separate security review explicitly approves them.
- Choose an explicit UX for the update control: hide/disable it, make it state
  that updates are managed externally, or route it to a project-owned,
  verified update mechanism.
- A project update mechanism must use fixed artifacts, cryptographic/hash
  verification, TLS validation, atomic/recoverable installation, and no
  arbitrary adjacent custom script execution.
- Test both the missing-updater and update-available paths without a printer
  before any hardware trial.
- Record the resulting behavior and any UI patch provenance. Do not import
  upstream installer code wholesale.

## Non-goals

- Enabling automatic network updates merely for convenience.
- Treating an upstream GitHub release as trusted without verification.
- Physical-printer testing in the initial audit.

## Historical proposed implementation

First confirm the current packaged binary's behavior when no adjacent
`update.sh` exists. Then prefer an upstream configuration or small, reviewed
patch that disables the action and presents a clear message. Revisit a
project-managed transactional update design only after the broader update and
rollback architecture is defined.

## Verification

Not run; no longer applicable because the GuppyScreen variant is retired.

## Hardware verification

Not run; no GuppyScreen image may be installed on a printer.
