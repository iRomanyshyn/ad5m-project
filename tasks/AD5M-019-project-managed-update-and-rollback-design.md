# AD5M-019 Project-managed update and rollback design

Status: proposed

## Goal

Design a recoverable, verifiable project update path for the AD5M stack. It
must replace the reviewed project release as one controlled unit; it must not
delegate update authority to HelixScreen or any other bundled application.

## Decision direction

Do not allow automatic application of updates in the first release. In
particular, upstream HelixScreen self-update stays disabled under AD5M-018.
The safe initial interaction is user-initiated: check, inspect the identified
project release and its notes, download/stage it, then explicitly approve its
application while the printer is idle.

Background notification or unattended application is not an initial goal. A
future notification check may be considered only as an explicit opt-in; an
unattended apply requires a separate decision after every criterion in this
task has production evidence. It is not justified merely because a newer UI
release exists.

## Context

An AD5M has constrained storage/RAM and runs the project alongside the vendor
host environment. A partial update can break the UI, web stack, Python
services, or boot integration while the printer remains physically useful.
Updating HelixScreen independently would also bypass the project's source
review, package closure, resource validation and compatibility testing.

The project already requires predictable upgrades and rollback. AD5M-015
therefore rejects in-application self-update before this task is completed,
and AD5M-018 requires the upstream update checker/self-updater to remain out
of the local-only variant.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- A release is immutable and identifies every installed artifact, source
  revision and compatibility/migration level. A mutable branch, GitHub latest
  asset or unauthenticated manifest is not an update target.
- Verify a project-controlled signature and artifact digest before staging; do
  not treat transport TLS, a release-page URL or a checksum served beside the
  artifact as sufficient authority by itself.
- Keep the current release bootable until the candidate has been completely
  staged and verified. Define the exact storage layout and atomic switch
  primitive from the actual AD5M install layout; do not assume A/B slots exist.
- Apply only after an explicit local user confirmation and only when the
  printer is genuinely idle. Block application during print preparation,
  printing, pause/recovery, firmware flashing, calibration or an uncertain
  printer state. An update must never override a print-safety state merely to
  meet a schedule.
- Preserve documented per-printer user state, including network identity,
  Moonraker database/authentication, G-code, HelixScreen layout/consent and
  other explicitly selected configuration. Migrations must be versioned,
  idempotent and reversible where feasible.
- On first start after switching, perform a bounded health check of the
  essential services and record a machine-readable result. A failed health
  check, failed migration, missing runtime dependency or boot-loop must return
  to the previous known-good release without touching motion, heaters, MCU
  firmware or printer configuration safety limits.
- Make rollback available through a local recovery path that does not need
  Internet, GitHub, upstream HelixScreen or a working touchscreen.
- The update agent runs with the minimum necessary privilege, uses no shell
  text assembled from release metadata, and keeps audit logs and diagnostic
  state bounded on constrained storage.
- UI/application packages cannot update themselves, replace the updater,
  alter its trust root, or enable automatic apply through their own settings.

## Non-goals

- Enabling upstream HelixScreen self-update, a generic package-manager update
  or automatic `pip`/npm/Git recovery.
- Automatically updating Klipper MCU firmware, heater/motion configuration or
  any other safety-critical printer component.
- Requiring cloud access to install, roll back or recover a release.
- Designing telemetry consent or external support uploads; AD5M-018 owns
  those boundaries.

## Proposed implementation

1. Inventory the current installer, persistent paths, boot/service order,
   writable capacity and recovery mechanisms on a disposable image. Choose a
   concrete staged-release and atomic-switch layout from that evidence.
2. Define the project release manifest, signature scheme, offline public-key
   rotation policy and verification tool. Keep the verifier small and
   independently testable.
3. Implement check/download/stage/verify/apply/health-check/rollback as
   explicit state transitions with durable journals and power-loss tests.
4. Add a local UI that displays release identity, compatibility notes, required
   restart and rollback availability, then asks for confirmation. The UI may
   request the operation; it does not own trust verification or privileged file
   changes.
5. Add software-only fault injection for interrupted downloads, bad signatures,
   exhausted storage, interrupted switch, migration failures, service failures
   and rollback. Perform hardware acceptance only after an installable image
   exists and no print is active.

## Acceptance criteria

- A project release can be staged, signature-verified and applied only after
  local confirmation on an idle printer.
- Interrupted or invalid candidate releases leave the current release usable.
- A post-switch failure automatically returns to the previously verified
  release and explains the reason through a local recovery path.
- Documented persistent user state survives both update and rollback; no
  authentication database or telemetry consent is silently reset.
- No production path lets HelixScreen or another bundled application fetch and
  apply its own upstream release.
- Automated tests cover integrity, state transitions, fault injection and
  rollback separately from physical printer evidence.

## Verification

Not run. This is an architectural task; no updater has been created or
executed.

## Hardware verification

Not run.
