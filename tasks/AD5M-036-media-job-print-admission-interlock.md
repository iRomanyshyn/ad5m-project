# AD5M-036 Media-job print-admission interlock

Status: planned

## Goal

If an optional media add-on ever encodes a timelapse on the printer, prevent a
new print from starting until that job has completed or the owner explicitly
cancels it. The block must apply equally to browser, slicer and local command
paths; a disabled button alone is not sufficient.

## Context

The AD5M base image deliberately contains no FFmpeg or timelapse feature
(AD5M-009). A future add-on would run on a 128 MiB device where encoding can
compete with Klipper/Moonraker for CPU, RAM and I/O. Rendering only after the
previous print finishes does not eliminate the race in which a user or slicer
starts the next print before encoding ends.

## Required contract

1. **One owner-controlled media job.** A rendering job has a unique ID and a
   durable, bounded state: `queued`, `running`, `succeeded`, `failed`,
   `cancelled`, or `interrupted`. It records only non-secret media input/output
   identities, timestamps and a bounded failure class.
2. **Atomic admission lock.** The renderer obtains a process-held exclusive
   lock before it becomes `running`; a print-start request and renderer cannot
   both win a race. A stale pathname or UI flag is not a lock. Boot recovery
   must identify a dead/interrupted owner rather than block printing forever.
3. **Deny before motion.** While a job is `queued` or `running`, every
   supported print-start path fails before G-code execution or motion:
   Moonraker native APIs, OctoPrint-compatible slicer uploads/start requests,
   job-queue paths if enabled, and direct virtual-SD/G-code start paths. The
   rejected caller receives a clear busy response naming the media job and a
   cancel/status route.
4. **Defence in depth, not UI theatre.** UI controls may disable “Print”, but
   the enforcing admission point must be authoritative. If Moonraker alone
   cannot cover direct Klipper start commands, add a narrowly reviewed Klipper
   guard or declare that path unsupported; do not silently leave a bypass.
5. **No surprise termination.** Starting a print never kills an encoder and an
   encoder never cancels an active print. Explicit owner cancellation must
   terminate and reap the encoder, retain an observable result, release the
   lock only after reaping, and leave printer state untouched.
6. **Safe failure behaviour.** Media worker crash, timeout, power loss,
   malformed journal, or unavailable storage becomes a visible failed or
   interrupted media state. It must neither start a print automatically nor
   leave an indefinite lock. The owner can acknowledge/cancel the stale state
   through a local API/UI flow.

## Implementation boundary

- Do not add this machinery to the base image. It belongs solely to a reviewed
  optional media add-on.
- First map the pinned Moonraker and Klipper start paths and their exact error
  semantics. Do not assume that an HTTP upload endpoint covers `SDCARD_PRINT_FILE`,
  `M23`/`M24`, HelixScreen, or a direct Klipper socket client.
- Prefer upstream extension points. Any project-owned Moonraker/Klipper patch
  must be small, source-pinned, documented and separately security-reviewed.
- The lock/journal location must be outside temporary frame directories and
  must have bounded storage and a documented migration/removal path.
- This control does not authorize an encoder, FFmpeg package, USB executable,
  automatic download, or MCU operation.

## Required tests before hardware

- Unit/fixture tests for all state transitions, duplicate requests, parallel
  start-versus-render races, cancellation, worker crash, timeout and reboot
  recovery. Tests use a harmless mock worker, never FFmpeg or a printer.
- Endpoint tests covering each supported Moonraker/slicer start route and a
  negative direct-G-code route. Each must prove rejection occurs before a
  virtual-SD print starts.
- A regression test that confirms printing is admitted after successful,
  failed, cancelled and recovered-interrupted jobs, with an explicit owner
  acknowledgement where required.
- Resource tests with synthetic data to bound log growth, frame-path storage,
  process count and lock lifetime.

## Hardware verification

Not run. Any future hardware test starts only after the software contract is
implemented and includes a recovery-ready printer, an idle precondition, and
observation that no print begins while the media job is active.
