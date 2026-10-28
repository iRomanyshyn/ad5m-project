# AD5M-035 Load-cell diagnostic command memory safety

Status: in progress

## Goal

Remove the reachable unbounded copy in the fork-specific load-cell diagnostic
path and ensure a production image cannot use arbitrary G-code text as a raw
MCU peripheral command.

## Context

AD5M-030 established that the active mainboard firmware includes
`flashforge_load_cell.c` and the host registers
`FLASHFORGE_LOAD_CELL_TEST CMD=<text>`.  The host encodes arbitrary text and
passes its length via `flashforge_loadcell_test_cmd cmd=%*s`; the MCU then
copies that caller-provided length into a 32-byte queue buffer with `memcpy`
and no length check.  A long command can corrupt adjacent MCU memory.

The regular project macros use only `FLASHFORGE_LOAD_CELL_TARE`; this generic
test passthrough is not a required normal-printing feature.  No command was
sent and no printer was tested while recording this source-level finding.

## Requirements

- The production profile must not expose an arbitrary raw load-cell command
  passthrough.  Prefer removing `FLASHFORGE_LOAD_CELL_TEST` completely from
  both host and MCU production sources.
- If a separate, explicitly selected diagnostic profile is justified later,
  it must use a reviewed finite command allowlist and enforce the same
  size/encoding limit independently at the host and MCU boundaries.  A UI-only
  restriction or a single caller-side check is insufficient.
- Every MCU queue write must validate length before copying, leave queue state
  unchanged on rejection, and report a bounded error.  Do not silently
  truncate a command whose protocol meaning could change.
- Add software-only tests/static checks that reject restoration of the generic
  passthrough and exercise overlong/invalid diagnostic input without compiling
  or flashing a target MCU.
- Record the changed source revision and generated MCU artifact identities.
  A fixed source build is not physical proof: rerun targeted load-cell,
  probing, homing and recovery validation before flashing it.
- Align the final authorization model with AD5M-016/029: even safe diagnostic
  commands must not be available to unauthenticated LAN clients.

## Non-goals

- Changing force thresholds, calibration values, motion behavior or the
  proprietary load-cell protocol without their own safety evidence.
- Treating a bound check as an authorization, rollback or MCU-recovery design.

## Verification

2026-10-28: integration commit `9239027` added local Klipper patch
`0004-Remove-unsafe-loadcell-raw-diagnostic.patch`.  It removes the generic
host G-code and MCU command registration, and rejects overlong queue input
before `memcpy`.  `tests/validate-static.sh` asserts that the patch retains
both controls; `make validate` passed.  A `patch --dry-run --fuzz=0` pass
applied it to the exact locally inspected pinned Klipper source without
writing files.  This is a source-only control: no clean package build, MCU
execution or flash has occurred.

## Hardware verification

Not run.  Any regenerated mainboard firmware requires a recovery-ready,
idle-printer test plan after software validation.
