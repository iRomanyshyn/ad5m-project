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

## Hardware verification

Not run.
