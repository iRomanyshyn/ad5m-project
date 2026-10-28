# AD5M-027 Installer preflight before destructive replacement

Status: in progress

## Goal

Reduce the legacy installer's avoidable failure modes immediately: it must
reject a malformed compressed chroot or inadequate free space before deleting
an existing chroot and its host-init entry.

## Context

The current `flashforge_init.sh` previously removed `/etc/init.d/S00klipper_mod`
and `/data/.klipper_mod/chroot`, then checked the 512 MiB threshold and only
then attempted to decompress `chroot.tar.xz`.  A corrupted USB archive or
insufficient post-removal capacity could therefore turn a recoverable failed
update into loss of the currently bootable project chroot.

This task is a narrow guardrail, not the transactional updater promised by
AD5M-019.  It does not establish a signed release identity, preserve a prior
chroot through extraction/setup, or make MCU updating reversible.  In
particular, candidate setup currently mutates shared persistent state and may
invoke the MCU updater; calling this flow "rollback" would be misleading.

## Immediate change

- Run `xz -t` and a tar listing pass over `chroot.tar.xz` before changing the
  installed chroot.
- Evaluate the existing free-space threshold before any old chroot/init entry
  is removed.  This is deliberately conservative: an update may be refused
  where the legacy flow would have deleted the old chroot first.
- Statically assert the required command ordering without executing the
  target installer.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- An archive-integrity or free-space preflight failure must leave the existing
  chroot and host init script in place.
- The preflight must run entirely from the package already selected by the
  stock updater; it must not fetch code, trust a detached MD5 list, or execute
  USB-provided helpers.
- Do not characterize this change as update authentication, atomic staging or
  MCU rollback.  AD5M-019 remains the owning design task for those properties.
- Validation must cover shell syntax and reject a source ordering regression.

## Verification

Run `make validate` in `ad5m_enhanced_mod`.  This is source-only validation;
it must not execute the installer, mount media, build an image or access a
printer.

2026-10-28: `make validate` passed after the change, including the static
ordering assertion and shell syntax checks.  The installer runs on the
closed stock host, not the Buildroot chroot; the stock host has not yet been
inspected to prove `xz -t` support.  If that option is absent, this preflight
will fail before deletion (safe refusal), but it may block installation.  A
read-only stock-host capability audit is required before any use.

## Hardware verification

Not run.  Before using the installer on a printer, test its preflight failure
and success paths on a disposable target image and complete AD5M-019 and
AD5M-026 recovery evidence.

## Remaining work

1. Stage a candidate without replacing the known-good root, then define an
   atomic selector and independently journaled health-check/rollback state.
2. Split MCU flashing from candidate boot/health checks; no candidate may
   flash an MCU merely to prove application health.
3. Add a signed manifest and artifact inventory, eliminate arbitrary overlay
   copy, and test interruption/storage-failure cases under AD5M-019.
