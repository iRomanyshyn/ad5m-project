# AD5M-025 Removable-media host trust boundary

Status: in progress

## Goal

Ensure that inserting removable media cannot cause unreviewed code to execute
on the privileged Flashforge host, while designing any future recovery-media
workflow as an explicit, verifiable operation.

## Context

The source audit for AD5M-024 found a host-init path independent of the
chroot.  `build_overlays/common/etc/init/S00klipper_mod` scanned VFAT volumes
for `klipper_mod_*` marker names.  If it found `klipper_mod_debug`, it remounted
that medium writable, copied host logs to it and invoked
`/bin/sh /mnt/mod_debug.sh`.  The init script runs as root before the chroot
starts.  The marker and script had no signature, device identity, local
confirmation or allowlist, so any matching USB medium could execute arbitrary
host code at boot.

This is established from the project's own source; it does not rely on a
third-party repository or on physical-printer testing.  It is especially
important while USB topology and potential internal storage remain under
investigation: physical access or a permanent device must not implicitly grant
software update authority.

## Immediate remediation

- Remove the `klipper_mod_debug` / `mod_debug.sh` host-init path.
- Do not mount removable media writable or copy diagnostics merely because a
  marker is present.
- Remove the installer-time chroot USB-root overlay; the selected packaged
  chroot is the only setup payload until a signed, reviewed recovery/update
  format exists.
- Keep a static negative test that rejects this exact class of host-init
  execution or root-overlay regression.

The existing `klipper_mod_skip`, `klipper_mod_remove` and stock-vendor-update
markers are deliberately unchanged by this narrow remediation.  That preserves
their current recovery behavior but is **not** an endorsement of their trust
model: marker-triggered skip/remove can affect boot availability and must be
redesigned before a project release can rely on them.

## Current USB-path inventory (2026-10-28)

The following is a source-only inventory of project-controlled paths.  It
does not establish what the closed stock host does after its own update handoff.

| Path | Current trigger and authority | Security consequence / owner |
| --- | --- | --- |
| Host `S00klipper_mod` | Before chroot startup it scans `/dev/sda`/`sdb`, partitions 1–4 and raw devices, mounting VFAT read-only. `klipper_mod_skip` restores stock MCU firmware and skips the chroot; `klipper_mod_remove` also removes the mod. An `Adventurer*.tgz` or `flashforge_init.sh` name makes it defer to the stock boot/update flow. | No USB-provided script is executed after the immediate remediation, but filename-only skip/remove remains an unauthenticated availability/destructive control. The broad vendor-update detection is a handoff, not project verification; redesign under this task and AD5M-019. |
| Runtime `automount.sh` via mdev | `sd[a-z].*` hot-plug events mount VFAT read-only under `/media/<MDEV>` and replace the `gcodes/usb` symlink. | Merely inserting media does not execute its contents. Selecting an imported G-code file is a separate printing-trust action; identity, multi-device behavior and any persistent/internal mount remain AD5M-024 work. |
| Project install archive | The stock update mechanism runs the archive's `flashforge_init.sh` as a privileged installer. It tests and lists the compressed chroot and checks the legacy free-space threshold before deleting the active chroot, then unpacks the supplied root filesystem. | An install archive is already a privileged external-code channel. The preflight is only a corruption/capacity guard: it has no manifest/signature verification, staging or rollback, and cannot be a project update mechanism; AD5M-019 owns its replacement. |
| Former `klipper_mod` installer overlay | Before 2026-10-28, `S00klipper_mod setup()` bind-mounted host `/mnt` into chroot `/media`, and `install_steps.sh` copied `/media/klipper_mod/*` into `/`. | Removed. The base installer no longer passes the USB medium into the chroot or applies an arbitrary root overlay. A future signed recovery/update format remains AD5M-019 work. |
| Install/uninstall logs | The privileged installer/uninstaller redirects logs to `/mnt` if the vendor has mounted a matching USB device there. | This is a diagnostic data-export path, not code execution, but logs may contain sensitive details. A future explicit export flow must redact secrets and identify its destination as required above. |

No project-controlled USB path was found that automatically sources or
executes a file from `/media` or `/mnt` after the removed debug path.  That is
not evidence that the vendor's opaque update dispatcher verifies arbitrary
archives, nor that the current installer is safe to use as a release updater.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- No boot-time host script may interpret, source or execute a program stored
  on removable media merely because its filename or a marker exists.
- Normal removable-media import stays read-only unless a separate,
  user-requested operation needs writes.  Such an operation must have bounded
  behavior, clear user-facing purpose and no code-execution side effect.
- A future diagnostic export must generate its data locally, require an
  explicit local action, avoid exporting secrets by default, and write only
  the selected artifact to an already identified destination.  It must not use
  a USB-provided helper script.
- A future recovery/update medium is subject to the signed-manifest, payload
  inventory, compatibility and local-confirmation requirements of AD5M-019.
  Presence of a file, partition label, USB serial number or a physical internal
  mount alone is not authority to apply it.
- Redesign or retire marker-driven `skip` and destructive `remove` behavior
  under a separately reviewed recovery-state model.  In particular, define
  physical/local confirmation, precise state transitions, logging, failure
  handling, and how a lost or malicious USB device cannot repeatedly alter
  boot behavior.
- Test the host-init policy without executing target code or requiring a
  printer.  Later test the approved recovery design with hostile filenames,
  malformed VFAT media, unexpected removal, power interruption and a device
  inserted at boot.

## Non-goals

- Treating removable media as a trusted source merely because it is internally
  mounted, has a familiar label or was previously used with the printer.
- Removing stock-vendor update or recovery behavior without an evidence-backed
  replacement and a user-approved transition plan.
- Enabling a project updater, automatic update, or USB-backed swap.  Those
  remain respectively AD5M-019 and AD5M-006 work.

## Verification

Software-only validation must confirm that the production host init script
contains no `klipper_mod_debug`, `mod_debug.sh`, shell execution or shell
sourcing from `/mnt` or `/media`, and that a deliberately unsafe fixture is
rejected.  It must also run shell syntax checks through the project
static-validation target.

2026-10-28: `make validate` in `ad5m_enhanced_mod` passed after the immediate
remediation.  It ran host lint, the production-policy check, its unsafe
fixture, shell syntax checks and the existing static checks.  No image build,
emulator or target code execution was performed.

## Hardware verification

Not run.  The remediation removes a host-init branch but does not establish
behavior on stock firmware, all USB devices or a future recovery release.

### 2026-10-28 installer-overlay removal

The project removed the `S00klipper_mod setup()` bind mount of stock-host
`/mnt` into chroot `/media` and the chroot `install_steps.sh` copy of
`/media/klipper_mod/*` into `/`. The installer archive remains the selected
stock-host input, but it no longer grants an additional arbitrary USB-root
overlay inside the extracted chroot. `make validate` now rejects restoration
of either half of this path. This is source-only verification; no installer,
USB device or printer was run.

## Remaining work

1. Inventory every host-side and chroot-side path that reacts to a USB marker,
   archive, label or mount event, including the vendor update handoff.  The
   project-controlled baseline above is complete only for the checked source;
   validate it against stock-host behavior before design decisions.
2. Decide whether skip/remove survive, and if so define a local, authenticated
   recovery state machine that cannot be activated by arbitrary inserted media.
3. Design a separately authorized diagnostic-export UX and a signed,
   offline-verifiable recovery format aligned with AD5M-019.
4. Run hardware-in-the-loop tests only after that design and test plan are
   approved; do not insert untrusted media on a printer just to test it.
