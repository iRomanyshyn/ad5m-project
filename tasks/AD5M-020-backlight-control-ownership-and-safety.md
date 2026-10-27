# AD5M-020 Backlight control ownership and safety

Status: proposed

## Goal

Give the AD5M display-backlight control path a project-owned, auditable
implementation and preserve the existing user-facing brightness controls
without importing historical community-mod code or making unchecked display
ioctls part of HelixScreen integration.

## Context

The current `lite` configuration selects the external Git package
`python-ff-ad5m-backlight` at immutable commit
`3cd9a0898e51a18e83976c7cab29037f8364f406`.  Its installed `backlight`
command is used at boot by `S90welcome` and by the `LCD_ON`/`LCD_OFF` Klipper
macros.  HelixScreen will also need a reviewed backlight contract, but must
not infer device paths or ioctls from a third-party package without a separate
AD5M hardware review.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Preserve an explicit, bounded brightness interface for existing macros and
  boot code, including an unambiguous off operation.
- Establish the authoritative AD5M display-kernel interface and the exact
  semantics of each operation before sending a newly implemented ioctl to
  hardware.
- Keep the production command local-only: no network client, updater,
  telemetry, shell evaluation, or runtime dependency download.
- Prefer a small project-owned implementation with fixed source and licensing
  over a continued third-party Git recipe, but do not copy code until its
  origin and licensing implications are reviewed.
- Ensure invalid input fails without modifying display state; retain a
  separately recorded physical-display test for every brightness transition.
- Do not change or exercise the hardware path merely as part of a source or
  package review.

## Non-goals

- Changing the display timing, device tree, PWM ownership, touchscreen
  calibration, or power sequencing.
- Claiming a host-side mock validates `/dev/disp` behavior.
- Adding display control to a network API or remote-control service.

## Implementation notes

### 2026-10-27 static inventory

The existing package is a fixed Git commit, GPL-3.0, and contains one
installed Python module plus the `backlight` entry point.  Its runtime module
imports only `argparse`, `fcntl.ioctl` and `struct`; static inspection found
no HTTP client, socket, downloader, subprocess or shell-execution path.  A
developer-only `buildall.sh` contains a PyPI upload command, but the Buildroot
recipe uses the normal setuptools package path and does not invoke that
helper.  This is limited static evidence, not a general endorsement of the
external source.

The module opens `/dev/disp` and uses these numeric operations:

| Operation | ioctl number | Existing caller |
| --- | --- | --- |
| set brightness | `0x102` | `backlight 1..255` or a percentage |
| get brightness | `0x103` | `backlight` with no argument |
| enable backlight | `0x104` | before non-zero brightness is set |
| disable backlight | `0x105` | `backlight 0` |

It converts `0%..100%` to `0..255` and rejects values outside that range.
`S90welcome` deliberately calls `backlight 0%` then `backlight 50%`, because
the current implementation documents an uninitialised/inverted initial
brightness state.  The Klipper shell-command extension splits arguments with
`shlex` and calls `subprocess.Popen` without a shell, so the present macro
does not add shell interpolation; the replacement must retain this property
and independently validate its sole brightness argument.

The source README credits discovery of these ioctls to historical community
work and links to that history.  Ideas or observable interface facts may be
used as research input under ADR-0004, but that attribution is not a reason
to retain the package or transplant its implementation wholesale.

### 2026-10-27 scoped vendor-reference check

The official Flashforge reference was checked at the already-recorded commit
`6f479ea5f23c1f69d569cc5d567e14707e97b7b2`, under the read-only procedure in
AD5M-014.  Its tree is a Klipper source tree; it contains generic Klipper
display/PWM code but no Linux kernel tree, device tree, display-driver headers
or identified host-side display-control implementation.  A text search for
`/dev/disp` and `DISP_LCD_` returned no match.  The literal `0x102` occurs
only in unrelated MCU/CRC headers, not as a display ABI declaration.  The
repository's `fw-Ad5m` directory contains opaque firmware artifacts, which
are not valid substitute evidence for this task and were not executed or
accepted.

Therefore this vendor repository does **not** establish the `/dev/disp` ABI
or validate the four existing numeric ioctl values.  A future implementation
needs separately published kernel/UAPI source or headers plus a controlled
physical-display review.  Until then, treat the current values as an observed
compatibility constraint, not an authoritative interface to reimplement.

### Candidate direction

After the vendor kernel/UAPI and a physical AD5M confirm the interface, add a
minimal project-owned command in the external tree, with the same intentionally
small CLI contract: no argument reads the current value; `0` disables; a
strict decimal value in the reviewed range sets brightness.  A tiny C command
is a candidate because it avoids introducing a Python package for one ioctl
interface, but it must use reviewed target headers and explicit structure
layout rather than copied magic constants.  Keeping a minimal Python command
is also acceptable if its source, package ownership and no-network runtime
properties are equally explicit.  Choose after measuring image size and
service integration; neither option should be implemented from the current
module by mechanical copying.

Until the replacement passes source, package and physical-display review,
leave the current pinned package unchanged.  HelixScreen's first package must
not add PWM auto-export or make direct brightness writes of its own; it should
consume the reviewed project command only after this task is complete.

## Acceptance criteria

- The selected implementation, source identity, license and target interface
  are recorded without reliance on historical community source code.
- Its command accepts only the documented brightness inputs and cannot execute
  arbitrary commands or contact the network.
- The existing boot and Klipper macro callers have a reviewed migration or
  compatibility path.
- Software-level argument and error tests are recorded separately from a
  physical AD5M test of read, off, minimum, normal, maximum and reboot
  behavior.
- HelixScreen does not duplicate the backlight-control path.

## Verification

Static inspection of the installed package source and current callers only.
No command was executed and no display, printer or network setting changed.

## Hardware verification

Not run.

## Remaining work

1. Locate the relevant vendor kernel headers/source and establish the display
   ioctl ABI and ownership of `/dev/disp`.
2. Review the existing package's pinned source and license provenance against
   the project source policy, then choose replacement or narrowly justified
   retention.
3. Design the project-owned command and build recipe without copying
   third-party implementation details blindly.
4. Add host-side argument/error tests and package checks.
5. Run the separately approved physical display sequence before migrating boot
   or HelixScreen callers.
