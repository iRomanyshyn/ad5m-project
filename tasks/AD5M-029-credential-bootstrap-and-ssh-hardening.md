# AD5M-029 Credential bootstrap and SSH hardening

Status: proposed

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

## Verification

Not run.  AD5M-028 source inspection only established the current shared
credential; no authentication behavior was changed or tested.

## Hardware verification

Not run.
