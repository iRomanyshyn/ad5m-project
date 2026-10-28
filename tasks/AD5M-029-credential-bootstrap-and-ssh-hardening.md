# AD5M-029 Credential bootstrap and SSH hardening

Status: in progress

## Goal

Eliminate the shared `root:klipper` image credential and define a recoverable,
per-printer first-access flow that does not silently open printer control to a
LAN.

## Context

AD5M-028 confirmed that the shared credential is both published in the legacy
README and configured in the image through
`BR2_TARGET_GENERIC_ROOT_PASSWD="klipper"`.  The device also runs LAN-facing
services and persists selected SSH/network state.  Removing the password
without a bootstrap/recovery design could lock out the owner; retaining it is
not acceptable for a releaseable image.

Moonraker's browser/API identity is a related but distinct boundary.  This
task must align with AD5M-016 rather than assuming an SSH password solves
browser authorization.

## Requirements

- Never ship a shared, predictable root password or packaged SSH private key.
- Define a per-printer bootstrap mechanism with clear local ownership proof,
  bounded lifetime and safe retry/recovery behavior.  It must work without
  cloud, a third-party relay or an external network service.
- Preserve or rotate existing authorized keys deliberately across a project
  update; do not overwrite network/SSH identity merely to reset access.
- Limit root remote access to the minimum required transport and audience;
  document password/key policy, host-key handling and failure recovery.
- Integrate the first-user flow with AD5M-016's Moonraker authentication
  bootstrap without a temporary LAN-wide trusted-client exception.
- Provide static configuration checks and target-level tests before release;
  do not test credentials on a production printer first.

## Non-goals

- Creating a cloud account, public remote-access relay or a universal recovery
  password.
- Changing motion, heater, MCU or update logic.

## Source constraints and alpha design (2026-10-28)

The current base configuration explicitly supplies `root:klipper` and enables
Dropbear.  The installer persists `/etc/dropbear`, `/root/.ssh` and the
Moonraker database below the per-printer static-data boundary.  This makes
host keys, authorized keys, normal Moonraker users and its system API key
durable across a chroot replacement, but it does not make the initial state
safe.

The pinned Moonraker source confirms that the first normal user is created by
its normal user API and that `force_logins` takes effect only once such a user
exists.  Therefore neither an empty database nor the system API key can be
treated as a first-owner proof.  The current generic Dropbear init script also
starts a network daemon before the existing Klipper/Moonraker services; it
needs an explicit profile rather than an inherited default password policy.

### Selected direction for the personal alpha

- Ship no root password and no packaged private key.  Remote administration is
  key-only after bootstrap; a release profile must explicitly verify the
  selected Dropbear password/root-login semantics rather than rely on defaults.
- The initial owner establishes access only through either a verified local
  console procedure (expected to be UART in the later physical session) or a
  per-device, operator-supplied public key consumed from a local ignored build
  input.  The latter is permitted for the owner's personal alpha only; it must
  never become a committed release asset or a public installation convention.
- Before a key and normal Moonraker user exist, do not expose a temporary
  unauthenticated LAN setup route.  The bootstrap helper, if any, must be
  invocable only from the verified local path, must create its own bounded
  state, and must not print a reusable secret into ordinary logs.
- Retain the existing persistent SSH/Moonraker paths deliberately, with
  documented permissions and backup/redaction rules.  Rotation/revocation
  must require local owner authorization and must not silently delete working
  keys or users during update/rollback.

### Open implementation proof

Before modifying the image, verify against the pinned Buildroot/Dropbear and
Moonraker sources that the selected configuration: locks password access while
allowing the intended key path, starts no public listener before bootstrap,
creates the first Moonraker user without direct database edits, and remains
recoverable after a failed bootstrap.  Exercise this on a disposable
non-printer rootfs/mock before the AD5M.  The actual console capability and
local ownership ceremony are P4 hardware evidence under AD5M-031/026.

## Verification

2026-10-28: source-only inspection established the current root-password,
Dropbear and persistent-state boundaries, and the pinned Moonraker first-user
constraint.  No authentication behavior was changed or tested.

## Hardware verification

Not run.
