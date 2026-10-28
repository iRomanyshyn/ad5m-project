# AD5M-034 MCU update result accounting

Status: proposed

## Goal

Make the legacy MCU-update result unambiguous: never record a successful
firmware identity unless the independently identified mainboard and eboard
operations each completed successfully, and never treat that result as a
replacement for recovery/rollback.

## Context

Static inspection of `build_overlays/common/usr/libexec/mcu_update.sh` on
2026-10-28 found that `update()` and `update_stock()` launch mainboard and
eboard flash commands in the background, then use a bare `wait` before writing
the persistent `.klipper_mcu_version` marker.  The script does not retain each
process ID or evaluate each named command.  A bare shell `wait` is not proof
that both flashing operations succeeded, so the persistent marker can obscure
a partial failure and suppress a later retry.

This is a concrete control-flow issue in the current legacy path, not an
assertion that a printer has failed to flash.  The broader design remains
unsafe for project updates because it performs automatic MCU flashing from a
version marker; AD5M-019 owns removal of that authority from normal updates.

## Requirements

- Keep a distinct PID/result for each flash operation and write success state
  only after both explicit success results are known.
- On any failure, persist a bounded failure state that identifies the failed
  side without falsely claiming a safe/consistent firmware pair.  Do not
  loop-flash automatically at every boot.
- Do not run mainboard and eboard operations in parallel unless the exact
  vendor tools, power budget and hardware behavior have evidence that this is
  safe.  The conservative design should prefer a deliberate, observable
  sequence.
- Separate current firmware identity, attempted artifact identity and result
  state.  A Klipper host version string is insufficient evidence that both MCU
  images are correct.
- Preserve the ability to recover to the compatible stock path only after its
  versions, tool behavior and board-specific recovery route are verified.
- Add shell/static tests for exit-status handling and source ordering.  Do not
  call vendor flash tools from tests.
- Reconcile the final behavior with AD5M-019, AD5M-026 and AD5M-030.  A narrow
  fix must not be represented as transactional updates or hardware recovery.

## Verification

Not run.  The finding is source-only; no target script or flash utility was
executed.

## Hardware verification

Not run.  A final design requires a disposable, recovery-ready hardware test
with independent evidence for both MCU identities.
