# AD5M-030 Klipper source and safety-patch audit

Status: in progress

## Goal

Establish a reviewable provenance and safety delta for the pinned Klipper
source and each local patch before its host code or generated MCU firmware is
treated as releaseable.

## Context

The project package currently retrieves commit
`fc2aaae692b5f703420915390a3dcb8cd514c384` from `xblax/klipper`.  It also
applies local patches, including a TRSYNC timeout relaxation, and builds
mainboard/eboard firmware from configured targets.  A Git commit pin is not a
substitute for reviewing the fork's divergence from official Klipper,
Flashforge-specific behavior, patch provenance or physical safety impact.

This is not an allegation about the fork or its authors.  It is a required
control for any privileged, safety-relevant source input under ADR-0001 and
ADR-0004.

## Requirements

- Identify the exact relation of the pinned commit to official Klipper and
  record a minimal, classified delta.  Do not merge, rebase or bulk-import
  history as an audit shortcut.
- Attribute every project-local patch, state its intended behavior and review
  its changed code paths.  For timing, probing, load-cell, heaters, motion,
  serial transport, MCU build/flash or power behavior, require a separate
  safety argument and hardware-validation plan.
- Compare AD5M-specific behavior against the vendor reference only for named,
  minimal questions at pinned revisions; do not treat vendor source as an
  automatic replacement.
- Produce a reproducible source/component record with commit IDs, licenses,
  patches and generated MCU artifact identities.
- Do not swap the source, relax/restore a timeout, or flash an MCU as part of
  this audit.  Any change needs an approved implementation task.

## Verification

### Initial source-only package/patch pass (2026-10-28)

The following observations used only tracked source plus an ignored local
developer build directory as a **non-release** inspection aid.  No generated
artifact was installed, flashed, executed on a printer or accepted as a
reproducibility result.

- The configured `ffad5m` MCU inputs select `N32G455xE` for `mainboard` and
  `N32G455xC` for `eboard`, both at 144 MHz with USART1 at 230400 baud.  This
  is build configuration evidence, not confirmation of the hardware fitted to
  any particular printer.  The obsolete-looking generic `stm32-mod` inputs
  select an STM32F103 profile and must not be assumed to describe the target.
- The three Klipper patches apply to distinct concerns: disable target-side
  C-helper freshness rebuilding, provide a local build-version fallback, and
  change `TRSYNC_TIMEOUT` from 25 ms to 100 ms.  The first two affect build
  identity/runtime library freshness; the last changes homing communication
  timing and needs a hardware safety argument.  Patch commit metadata and a
  small diff are not a provenance or safety approval.
- The local build tree contains the expected post-patch `TRSYNC_TIMEOUT =
  0.100` and produced mainboard/eboard `.bin`, `.elf` and `.hex` files.  Its
  artifact hashes are deliberately not copied into a release record because
  that tree is ignored local output, not a clean reproducible build.
- The runtime updater starts mainboard and eboard flash commands in parallel
  then invokes `wait` without saved process IDs before recording the shared
  version marker.  A bare `wait` does not establish that both named flashing
  operations succeeded.  A failure can therefore be obscured by marker state
  and prevent a retry.  AD5M-034 owns this independent control-flow defect;
  it is not fixed as part of this audit.

Next source-only work is to establish the exact upstream/fork delta and
classify every changed safety-relevant path.  No source swap, timeout change
or firmware flash is authorized by this status change.

### Fork topology and delta inventory (2026-10-28)

Read-only Git comparison established the pinned commit's relationship to
official Klipper without merging or importing either history:

- pinned fork commit: `fc2aaae692b5f703420915390a3dcb8cd514c384`, tagged in
  that fork as `v0.13.0-285-gfc2aaae69-ffadm5`;
- official comparison head fetched for the audit:
  `020d25e7fed80393dc4f1ae1720bdab9d8c0a038`;
- merge base: `96c3ca160e881dbeef8ccbe80f804572b9170d8c`
  (`gcode: Fix out-of-order check for M112 when read from gcode pseudo-tty`);
- 565 official commits are absent from the pinned fork, while 32 commits are
  unique to the pinned fork.  The count is a topology fact at the recorded
  comparison heads, not a quality or security score.

The 32 unique commits modify only these nine paths:

| Class | Paths | Why it is a release gate |
| --- | --- | --- |
| Load-cell host and MCU protocol | `klippy/extras/flashforge_loadcell.py`, `src/stm32/flashforge_load_cell.c` | It polls a proprietary UART protocol, exposes force state, and can invoke pause or Klipper shutdown on a configured threshold. |
| TVOC host and MCU protocol | `klippy/extras/flashforge_tvoc.py`, `src/stm32/flashforge_tvoc.c` | It configures a second MCU UART and reports environmental readings; pin/UART assumptions require target evidence. |
| Nations MCU support | `src/stm32/Kconfig`, `src/stm32/Makefile`, `src/stm32/stm32f1.c` | The selected AD5M configs require N32G455 variants, clocking and bootloader behavior that official Klipper did not contain at the fork point. |
| Eboard boot/reset behavior | `src/generic/armcm_reset.c`, `src/stm32/stm32f1.c` | It changes `FIRMWARE_RESTART`/bootloader handling for the eboard; a reset path is safety- and recovery-relevant. |
| Virtual-SD event | `klippy/extras/virtual_sdcard.py` | It changes host event behavior consumed by the load-cell integration. |

Static inspection of the final pinned source also found two specific review
questions that cannot be dismissed as generic fork noise:

- `FLASHFORGE_LOAD_CELL_TEST` is registered as a G-code command and forwards
  arbitrary bounded text to the proprietary load-cell protocol.  A production
  policy must either omit it, restrict it to a deliberate diagnostic profile,
  or prove its authorization and safe command allowlist.  It must not become a
  convenient remote debug backdoor through a broadly authorized Moonraker.
- `FLASHFORGE_LOAD_CELL_SET_MAX_FORCE` changes the in-memory overload
  threshold/action (`shutdown` or `pause`).  The final access policy and
  macro/UI design must ensure that ordinary remote/browser clients cannot
  silently weaken this safety behavior.  Its intended physical threshold and
  action still require hardware validation.

This pass establishes scope, not correctness.  The remaining audit is a
line-by-line review of the four safety-critical code groups, minimal vendor
comparison for their protocol/boot assumptions, a deliberate disposition for
each diagnostic command, and a hardware test plan before any MCU artifact can
be releaseable.

### Active integration and bounded-command finding (2026-10-28)

The fork additions are active project inputs, not dead source: the `ffad5m`
mainboard build configuration enables both `CONFIG_FLASHFORGE_LOADCELL` and
`CONFIG_FLASHFORGE_TVOC`; the eboard configuration enables neither.  The
shipped base printer configuration instantiates both host components on `mcu`,
and production macros invoke `FLASHFORGE_LOAD_CELL_TARE` before manual probing
and bed-mesh calibration.  No configured `max_force` value overrides the host
component's source default of 900 g with `shutdown` action.  Those are source
facts only; they do not establish the correct force threshold, pin mapping or
safe target behavior.

A concrete source defect was found in the diagnostic route:

1. `FLASHFORGE_LOAD_CELL_TEST CMD=<text>` accepts arbitrary UTF-8 text in the
   host component and forwards its encoded length through an MCU `%*s`
   command.
2. `command_flashforge_loadcell_test_cmd()` passes that length to
   `enqueue_flashforge_command()`.
3. The latter copies `cmd_len` bytes into `struct queued_cmd::cmd_data`, a
   32-byte array, with no bounds check.

An overlong G-code value can therefore write beyond the MCU queue slot.  This
is a source-level memory-safety finding; it was not sent to a printer and is
not evidence of an observed hardware failure.  The debug command is not
needed by the regular shipped macros.  AD5M-035 owns its removal or a strictly
bounded diagnostic-profile redesign.  It is a release blocker for any image
that includes this MCU firmware.

### Remaining fork-path review — 2026-10-28

The following review used the final locally inspected pinned source as a
read-only aid.  A temporary fork clone was a partial/promisor checkout, so a
network-independent textual diff could not be completed for every historical
blob after its remote became unavailable.  The observations below are thus
from the active final source/configuration and commit metadata, not a claim of
a complete reproducible historical diff.

| Area | Active source behavior | Disposition |
| --- | --- | --- |
| TVOC host/MCU pair | The mainboard enables `CONFIG_FLASHFORGE_TVOC`; the host config creates `[flashforge_tvoc]` and exposes it through a `temperature_sensor`.  The MCU binds UART4 RX (`PC11`) at 9600 baud, parses 9-byte `FF 18` frames with a checksum, and publishes CO2, TVOC and HCHO values.  The host samples its last TVOC value every 0.5 s, reports a zero "target", and deliberately makes `setup_minmax()` a no-op. | No direct heater, motion or MCU-flash action was found in this code path, but pin/voltage/protocol assumptions and the use of a generic temperature-sensor interface are unverified.  Keep it out of safety-control decisions and require a target read-only sensor/protocol test before accepting a flashed artifact. |
| Eboard reset/bootloader | The checked eboard config selects `N32G455xC`, 144 MHz, a 64 KiB bootloader offset (`FLASH_APPLICATION_ADDRESS=0x08010000`); the mainboard selects `N32G455xE` with no bootloader offset.  For the eboard combination, `FIRMWARE_RESTART` writes backup-register magic `0x1234` then resets, while bootloader request is a bare reset because the code assumes the eboard bootloader's default behavior. | Do not alter, call, or treat this as a recovery guarantee before proving the exact board/bootloader behavior.  A failed assumption affects the ability to reconnect or reflash, so AD5M-026/034 own target evidence. |
| N32G45x support | The fork maps N32G45x to the STM32F1 family and defines clock, flash-size and boot-offset values used by the selected AD5M configs.  The configuration is evidence of intended target identity only. | Retain as a pinned fork-specific input.  Verify actual MCU markings, boot offsets, clocks and serial links during read-only hardware discovery before building an accepted MCU artifact. |
| Virtual-SD event | The fork emits `virtual_sdcard:load_file` after file selection and print-stat setup.  Project `md5_checker` and timelapse extensions subscribe to virtual-SD events; the change is an integration contract, not a peripheral protocol.  Commit metadata identifies the feature as `virtual_sdcard: emit events on file load` (2025-06-13). | Add a host-side fixture for event ordering, reset/cancel/error and duplicate file selection before changing/removing it.  It does not by itself justify any MCU change or hardware flash. |

### Test plan for the remaining paths

1. Add a pure host Klipper-event fixture for the virtual-SD load/reset/error
   contract and its `md5_checker`/timelapse consumers.  It must use synthetic
   files and no Moonraker endpoint or MCU.
2. Build the selected mainboard/eboard sources only after package provenance
   and clean-build gates are met; record config and artifact hashes, but do
   not flash them as a "test".
3. During the dedicated read-only hardware session, first verify controller
   markings, UART wiring/levels, the TVOC module's passive output, and the
   eboard's observed reset/boot behavior.  No sensor reading may drive a
   heater/motion decision, and no bootloader request is issued merely to
   identify it.
4. Only with recovery evidence may a separately approved operation exercise
   MCU firmware/restart behavior.  A successful host build or static source
   review never closes that gate.

## Hardware verification

Not run.
