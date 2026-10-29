# AD5M-019 Project-managed update and rollback design

Status: proposed

## Goal

Design a recoverable, verifiable project update path for the AD5M stack. It
must replace the reviewed project release as one controlled unit; it must not
delegate update authority to HelixScreen or any other bundled application.

## Decision direction

Owner-opt-in automatic updates are a product goal, not a feature to be
permanently withheld.  The initial personal-alpha interaction remains
user-initiated: inspect the identified project release and its notes,
download/stage it, then explicitly approve its application while the printer
is idle.  That gives the project a recovery baseline before unattended
operation is offered.

After the manifest, staging, health-check and rollback requirements below have
production evidence, the owner may select an update channel and opt into
automatic staging/application.  Automatic application must still wait for a
known idle state, use a signed immutable release, record an outcome, and roll
back on failure.  It is not justified merely because one UI component reports
a newer upstream version.

HelixScreen, Fluidd, Klipper and similar components may update automatically
only through that project-managed compatibility decision: either as a reviewed
release bundle or as a manifest-declared component with an explicitly verified
independent compatibility/rollback contract.  An embedded upstream updater
may report availability to the project updater, but must not download/apply
outside its authority.  MCU firmware and safety-critical printer
configuration are never automatic-update payloads.

## Context

An AD5M has constrained storage/RAM and runs the project alongside the vendor
host environment. A partial update can break the UI, web stack, Python
services, or boot integration while the printer remains physically useful.
Updating HelixScreen independently would also bypass the project's source
review, package closure, resource validation and compatibility testing.

The project already requires predictable upgrades and rollback. AD5M-015
therefore rejects independent in-application apply before this task is
completed, and AD5M-018 requires the upstream update checker/self-updater not
to bypass the project-managed release boundary.

### Current installer baseline audit (2026-10-27)

The current USB installer is an initial-install/reinstall mechanism, not a
candidate transactional updater. Static inspection of
`device_files/install/flashforge_init.sh` shows that it removes the active
host init entry and `/data/.klipper_mod/chroot` *before* checking free space or
unpacking the new `chroot.tar.xz`. It prints MD5 digests for logging but has no
embedded expected digest, signature verification, alternate release slot,
candidate health check or rollback selection. A space, archive or setup
failure after that deletion can therefore leave no prior chroot to boot.

AD5M-027 introduces a deliberately limited preflight guard before that
destructive step.  It does not alter this task's conclusion: archive and
storage preflight cannot make the legacy flow transactional after a candidate
starts changing shared state or flashing MCU firmware.

Its in-chroot setup deliberately has semantics that an application update must
not inherit. Earlier revisions copied an arbitrary USB overlay into `/` and
then invoked `S54mcu_update`, which flashed both MCUs on a retained
Klipper-version mismatch. Both normal-path behaviours were removed on
2026-10-28: the installer no longer imports a USB root overlay and S54 is a
no-op. The old version marker remains neither an MCU-artifact identity nor an
update authorization decision. Legacy stock-transition/uninstall flashing is
separately constrained by AD5M-034/026 and cannot become a background or
user-confirmed project updater.

There is useful recovery-related context, but not a release rollback system:
host `S00klipper_mod` can honor markers to boot stock once or remove the mod,
and restores stock MCU firmware before those actions. It binds `/data` into
the chroot. `install_steps.sh` keeps selected state under
`/mnt/data/.klipper_mod/static` (network identity, SSH keys, Moonraker
database and G-code), while the build post-processing stages fresh printer
configuration into each root filesystem. AD5M-019 must inventory and preserve
this boundary deliberately rather than assuming it is a complete migration
scheme.

A local developer `lite` bundle was inspected for scale only, not accepted as
a release: its embedded `chroot.tar.xz` was 56,606,892 bytes and its decoded
tar stream was 250,572,800 bytes. Tar-stream size is not a measurement of
target filesystem allocation, and a dirty local artifact is not release
evidence. It is nevertheless enough to reject an unmeasured dual-slot
assumption: the current installer's 512 MiB free-space threshold is checked
only after deleting the old chroot, while keeping an old root, a candidate root
and a staged compressed artifact can require substantially more capacity.
Measure the selected variant on the actual target layout before choosing an
atomic-switch strategy.

### Current persistence inventory (2026-10-27)

The current chroot setup moves only the following paths to
`/mnt/data/.klipper_mod/static` on first installation and makes the chroot
paths symbolic links back to them on later installations:

| Current path | Current purpose | Update-design consequence |
| --- | --- | --- |
| `/etc/hostname`, `/var/lib/iwd`, `/etc/network/interfaces`, `/etc/wpa_supplicant.conf` | device name and network identity | Preserve deliberately; never overwrite or export Wi-Fi secrets in a release artifact or diagnostic log. |
| `/etc/dropbear`, `/root/.ssh` | remote-access host and user keys | Treat as sensitive per-printer state. A candidate may use it only through the defined state boundary, and a release must not replace it with packaged keys. |
| `/root/printer_data/database` | Moonraker database, including the MCU-version marker and potentially authorization state | Migration must preserve authorization while being compatible with both the candidate and rollback release; the existing Klipper-version marker is not sufficient release identity. |
| `/root/printer_data/gcodes` | user G-code | Preserve without allowing an update manifest or migration to execute its contents. |

`/root/printer_data/config` is *not* in that static list. Build post-processing
creates it anew in every root filesystem from `printer_configs/`; the current
installer therefore does not establish that arbitrary edited printer
configuration survives a chroot replacement. Likewise, current scripts do
not establish a persistent HelixScreen layout/consent location. AD5M-019 must
define an explicit, minimal state schema for any configuration it promises to
retain, including ownership, permissions, migration version and backup/restore
rules. It must not make all of `/data` persistent by default.

The host boot script bind-mounts the whole `/data` partition into the chroot,
so a candidate release has access to this persistent state before it becomes
known good. The update design must validate a candidate before granting it
write access to migration state where possible, journal every migration, and
require forward *and backward* database/state compatibility until the health
check commits the switch. A rollback must not be forced to consume a state
format that only the failed candidate understands.

### Current package identity baseline (2026-10-27)

`build_scripts/build.sh` currently compresses `rootfs.tar` into
`chroot.tar.xz`, then creates a USB `.tgz` containing that file and all files
under `device_files/install`. Its filename derives from
`git describe --tags --always --dirty`; the Pro-named package is a byte copy of
the same archive. The later `checksums` helper emits an external `md5sums`
file. Neither the package nor the installer contains a signed manifest, an
expected digest, a release key identifier, a minimum/maximum compatible state
schema, or a complete component inventory. The install script merely prints
MD5 values for a log.

That baseline is useful for USB distribution but not for update authority. A
future immutable project manifest must be canonicalized before signing and at
minimum bind together:

- format/version and unique release identifier;
- supported machine identifiers and minimum vendor-host compatibility;
- every staged payload's filename, length and SHA-256 digest, including the
  privileged updater/recovery pieces rather than only the root filesystem;
- exact project source revision, reviewed Buildroot contract revision, selected
  variant/plugins and the full component/source inventory produced by the
  release build;
- persistent-state schema and migration/rollback compatibility range;
- an explicit declaration that the release contains no MCU firmware operation;
- signing key identifier, signature algorithm, signed creation time and an
  expiry/revocation policy that still permits offline recovery.

The installed verifier must reject duplicate fields, unrecognized critical
fields, path traversal, payloads not enumerated by the manifest, malformed
canonical encoding and a release that is validly signed but incompatible with
the device. A human-facing version string may be displayed, but it must not be
used as the authorization decision or rollback ordering on its own.

### Candidate health-check boundary (2026-10-27)

The current init order starts `S54mcu_update`, then Klipper (`S60`) and
Moonraker (`S65`), followed by the web servers and camera service. The Klipper
and Moonraker scripts fork their daemons and return immediately, so an init
exit status proves neither configuration loading nor Klippy connectivity.
More seriously, the current `S54mcu_update` will flash MCUs when its retained
Klipper-version marker differs. A candidate booted only to assess health must
therefore use an update-specific boot profile that explicitly excludes that
service and any equivalent automatic firmware action.

At the pinned Moonraker revision, its read-only `/server/info` response
reports `klippy_connected`, `klippy_state`, `failed_components` and
`missing_klippy_requirements`. These are suitable *inputs* to a local
readiness check after the project defines its final Moonraker authorization
profile; their exact expected values must be pinned to the release profile,
not guessed from a public LAN request.

The first-release health check must be bounded and read-only with respect to
the printer. At minimum it should verify the selected release identifier and
mounted paths, expected service executables/configuration, the controlled
Moonraker readiness result, and the selected UI process or local UI socket
when HelixScreen becomes part of the release profile. It must not send G-code,
home or move axes, heat, toggle power, restart firmware, start a print, upload
a file, contact the network, require a camera frame, or invoke MCU tools.
Network reachability, NTP, mDNS, an external telemetry receiver and optional
camera hardware are expressly not proof that a candidate is healthy and must
not prevent a safe local rollback.

The updater must persist a bounded machine-readable outcome containing the
candidate release ID, step, error class and rollback reason. It needs an
attempt counter/deadline independent of the candidate root so a reboot loop
or an unreadable candidate cannot retry forever. Only after all checks pass may
it mark the candidate known-good; otherwise the independent boot selector must
choose the prior known-good release before normal services run.

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
- Put the immutable release manifest and its signature inside the staged
  update, and have the small installed verifier validate the manifest's full
  payload inventory, compatibility and update policy before extracting or
  switching anything. A package filename, `git describe` label or detached MD5
  list is display/logging metadata, not authorization.
- Keep the current release bootable until the candidate has been completely
  staged and verified. Define the exact storage layout and atomic switch
  primitive from the actual AD5M install layout; do not assume A/B slots exist
  or that the current 512 MiB installer threshold can fund one.
- Apply only after an explicit local user confirmation and only when the
  printer is genuinely idle. Block application during print preparation,
  printing, pause/recovery, firmware flashing, calibration or an uncertain
  printer state. An update must never override a print-safety state merely to
  meet a schedule.
- Do not reuse the current destructive USB installer, its arbitrary overlay
  copy, or its version-marker-driven automatic MCU flashing as an update
  primitive. Project updates must treat MCU firmware as a separate,
  explicitly reviewed safety-critical operation.
- Preserve documented per-printer user state, including network identity,
  Moonraker database/authentication, G-code, HelixScreen layout/consent and
  other explicitly selected configuration. Migrations must be versioned,
  idempotent and reversible where feasible, with an explicit compatibility
  contract for both the candidate and previous release until health-check
  commit. Do not turn the whole `/data` partition into unmanaged persistent
  state.
- On first start after switching, perform a bounded health check of the
  essential services and record a machine-readable result. A failed health
  check, failed migration, missing runtime dependency or boot-loop must return
  to the previous known-good release without touching motion, heaters, MCU
  firmware or printer configuration safety limits.
- Run a candidate under an update-specific profile that excludes the current
  version-marker-driven `S54mcu_update` flow. The health check must use only
  release-pinned, local and read-only probes; it must not make camera, LAN,
  cloud, telemetry or time synchronization a prerequisite for keeping a
  release.
- Make rollback available through a local recovery path that does not need
  Internet, GitHub, upstream HelixScreen or a working touchscreen.
- The update agent runs with the minimum necessary privilege, uses no shell
  text assembled from release metadata, and keeps audit logs and diagnostic
  state bounded on constrained storage.
- UI/application packages cannot update themselves, replace the updater,
  alter its trust root, or enable automatic apply through their own settings.
  They may participate in a reviewed project-managed automatic-update channel
  once its bundle/component compatibility contract has evidence.

## Non-goals

- Enabling unreviewed upstream HelixScreen self-update, a generic
  package-manager update or automatic `pip`/npm/Git recovery outside the
  project update authority.
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
   restart and rollback availability.  Initially it asks for confirmation; a
   later owner-selected channel may authorize idle-only automatic application.
   The UI may request the operation; it does not own trust verification or
   privileged file changes.
5. Add software-only fault injection for interrupted downloads, bad signatures,
   exhausted storage, interrupted switch, migration failures, service failures
   and rollback. Perform hardware acceptance only after an installable image
   exists and no print is active.

## Acceptance criteria

- A project release can be staged, signature-verified and applied after local
  confirmation on an idle printer; after recovery evidence, an owner-selected
  channel may apply it automatically only while the same idle constraint holds.
- Interrupted or invalid candidate releases leave the current release usable.
- A post-switch failure automatically returns to the previously verified
  release and explains the reason through a local recovery path.
- Documented persistent user state survives both update and rollback; no
  authentication database or telemetry consent is silently reset.
- No production path lets HelixScreen or another bundled application fetch and
  apply its own upstream release outside the project-managed updater.
- No project-update path invokes `S54mcu_update`, accepts an arbitrary USB
  root overlay, or deletes the active chroot before the candidate is verified
  and recoverable.
- Automated tests cover integrity, state transitions, fault injection and
  rollback separately from physical printer evidence.

## Verification

Not run. This is an architectural task; no updater has been created or
executed.

## Hardware verification

Not run.
