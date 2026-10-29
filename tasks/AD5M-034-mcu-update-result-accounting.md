# AD5M-034 MCU update result accounting

Status: in progress

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

The historical automatic paths were broader than the service name suggested.
The normal chroot boot/install triggers were removed on 2026-10-28: the
base-image `S54mcu_update` slot is now a no-op and `install_steps.sh` does not
start it. Host `S00klipper_mod` still calls `update_stock` before its legacy
stock skip/remove paths when `check_stock` does not see the literal marker
`stock`; the uninstall archive also invokes `update_stock`. Those are legacy
stock-transition/recovery paths, not a release updater, and remain pending
AD5M-026 evidence. Removing or rewriting them without a demonstrated stock
boot/recovery route could itself create a brick path.

Neither a version-marker mismatch nor a skip/remove marker is proof of a
deliberately selected, compatible mainboard-and-eboard artifact. Normal
candidate boot, installation and project update must never turn either signal
into an MCU operation.

## Proposed MCU-operation contract

This is a design contract, not an implementation authorization.

1. **No automatic MCU flashing.**  Boot, install, uninstall, a version-marker
   mismatch, USB insertion, recovery-mode selection and application health
   checks must not invoke a vendor flasher.  A candidate release explicitly
   declares whether it contains *no* MCU operation; that is the only allowed
   first-alpha setting.
2. **Explicit request and immutable input.**  Any future flash begins only
   from a local, user-confirmed operation after the printer is idle and power
   stability/recovery prerequisites are checked.  The request records the
   requested board (`mainboard` or `eboard`), artifact SHA-256, artifact
   provenance, vendor-tool identity, intended action and release ID.  A
   display version or Klipper host revision alone is never an artifact ID.
3. **One board at a time.**  The controller operations are serialized.  The
   second operation cannot begin until the first named process has exited
   successfully and its result has been durably recorded.  Parallel flashing
   is prohibited unless a separately reviewed hardware experiment establishes
   it as safe for this exact board/tool pair.
4. **Durable, truthful states.**  State is outside a candidate chroot and is
   written before each irreversible transition.  The minimum states are
   `prepared`, `mainboard-running`, `mainboard-tool-succeeded`,
   `eboard-running`, `eboard-tool-succeeded`, `complete`,
   `failed-mainboard`, `failed-eboard`, and `interrupted`.  Each includes the
   operation ID and immutable input identities.  `*-tool-succeeded` means
   only that the named vendor command returned success; it is not a claim that
   the board is running the expected firmware.
5. **No implicit retry or completion.**  Loss of power, missing/invalid
   journal, process termination, timeout, nonzero exit, or unexpected boot
   state becomes a visible `interrupted`/`failed-*` condition.  It neither
   writes `stock`/a version marker nor re-flashes at the next boot.  A later
   operator must inspect the journal and choose an evidenced recovery route.
6. **Independent post-operation evidence.**  Only a board-specific,
   read-only verification method established by AD5M-026 may transition a
   complete pair from "tool reported success" to "verified".  Until then, an
   artifact/version marker is retained as an attempt record, not firmware
   truth.
7. **Bounded records and safe privacy.**  The journal contains IDs, hashes,
   timestamps, selected side and sanitized exit/error class.  It excludes
   Wi-Fi credentials, authorization tokens, raw USB paths supplied by users
   and unbounded tool output.  Retention/rotation must not consume unbounded
   `/data` space.

## Required transition table

| Transition | Preconditions | Durable record before action | Permitted successor |
| --- | --- | --- | --- |
| request → `prepared` | local confirmation; signed/identified artifact; printer idle; recovery evidence available | operation ID, board order, hashes, tool identity | first board `*-running` |
| `mainboard-running` → `mainboard-tool-succeeded` | exact saved mainboard PID exited 0 | mainboard exit class/result | eboard `*-running`, or stop for explicit verification where the design requires it |
| `eboard-running` → `eboard-tool-succeeded` | exact saved eboard PID exited 0 | eboard exit class/result | `complete` only after required pair evidence |
| any running state → `failed-*`/`interrupted` | nonzero exit, timeout, reboot, signal or unreadable state | failure class and affected side if known | no automatic successor |
| `complete` → `verified` | approved board-specific read-only evidence | verification method/result | normal operation; never an auto-flash trigger |

The old shared `.klipper_mcu_version` string cannot represent this table.  It
may be read during migration for diagnostics, but it must not authorize an
operation or suppress a safety decision.

## Implementation and test boundary

- Replace the legacy automatic-service call sites before introducing any new
  flashing implementation; do not make a better `wait` routine reachable from
  an automatic boot path.
- Use separate saved PIDs and separate waits if a future implementation starts
  a process.  The test suite must prove that each named nonzero result, signal
  and timeout produces the correct durable failure state; it must never call
  `NationsCommand`, `IAPCommand`, serial devices or a printer.
- Exercise journal parsing, malformed/missing state, stale lock handling,
  duplicate request rejection, error-output bounds and reboot/interruption
  transitions with fixtures on the development host.
- Do not manufacture a `verified` result in unit tests.  Hardware verification
  remains a separate evidence-gathering operation under AD5M-026.

## Requirements

- Implement the proposed contract or a reviewed equivalent; do not retain
  automatic flashing as the default behavior.
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

### 2026-10-28 immediate automatic-path removal

The base-image `S54mcu_update` init slot is now an explicit no-op and
`install_steps.sh` no longer invokes it. This removes the two normal-path
triggers that previously flashed both controllers because a Klipper host
version marker differed: ordinary chroot boot and install completion.

This intentionally does **not** implement a new MCU flasher or claim that a
candidate is compatible with stock controller firmware. The legacy explicit
stock/uninstall path remains untouched pending AD5M-026 recovery evidence:
altering it without proof that stock boot works with the retained controller
state would trade one unevidenced recovery risk for another. It must not be
used as an automatic update mechanism.

`make -C ad5m_enhanced_mod validate` statically verifies that `S54mcu_update`
cannot call an update action and `install_steps.sh` cannot start that service.
This was source-only validation; no target script, flash utility, serial
device, controller or printer was executed.

### 2026-10-29 remaining legacy-path boundary

Source inspection reconfirmed that the only remaining shipped callers of
`update_stock` are the host skip/remove transition and the uninstall archive.
They are still capable of invoking the vendor tools automatically once that
separate stock-transition action has been selected. They must not be reused
for normal install, boot, rollback health checks or project release updates.
The current script still starts its two board operations in parallel and uses
a bare `wait`; its marker result is therefore not trustworthy after partial
failure. The next implementation slice must either replace this legacy path
with the serial, journaled contract above and host-only fake-tool tests, or
retain it as an explicitly unsupported recovery exception until AD5M-026 has
evidence. It must not silently claim the existing marker is a verified MCU
identity.

## Hardware verification

Not run.  A final design requires a disposable, recovery-ready hardware test
with independent evidence for both MCU identities.
