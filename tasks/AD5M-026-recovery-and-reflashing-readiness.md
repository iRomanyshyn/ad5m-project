# AD5M-026 Recovery and reflashing readiness

Status: proposed

## Goal

Establish a board-revision-specific, minimally invasive recovery capability
before installing a project image on a printer, so a failed host or MCU update
does not depend on guesswork, undocumented connectors or a second working
printer.

## Current evidence

The project has two distinct firmware domains.

| Domain | Existing in-system path | What it can recover | What it cannot prove |
| --- | --- | --- | --- |
| Host / Linux image | The stock USB updater runs `flashforge_init.sh`; the project package is an archive for that path. | A printer whose stock host still recognizes and applies a compatible USB update. | An unbootable host, the integrity of an arbitrary package, partition layout, or a rollback after interrupted extraction. |
| Printer MCUs | `mcu_update.sh` invokes the stock-host `NationsCommand` on `/dev/ttyS5` for the mainboard and requests the eboard serial bootloader over `/dev/ttyS1` before invoking `IAPCommand`. | Restoring stock or project MCU firmware while the compatible host environment can run the vendor utilities and communicate with both MCUs. | Recovery if the host cannot boot, serial path/device firmware is damaged, or the board-level programming pads are unknown. |

The T113-S3 SoC has an immutable BootROM with FEL recovery support: its manual
describes FEL as a USB boot/recovery facility able to write external NVM, and
the maintained `sunxi-fel` tool communicates with that BootROM over a USB OTG
interface.  This establishes a possible lower-level recovery class, **not** an
AD5M procedure.  The project has no verified AD5M OTG connector, FEL-entry
method, storage layout, full-image dump or board-revision mapping.  The user
USB-A port must not be assumed to be OTG/device-capable.

The MCU configuration identifies STM32F103-class targets.  A genuine
ST-LINK/J-Link-class SWD probe could be a last-resort programming/debug tool
only after the actual SWD pads, voltage level and target ownership are
verified.  It is not appropriate to attach a probe based on MCU family alone.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod, only for evidence-backed tooling or release checks

## Requirements

- Before the first project installation, retain an official compatible stock
  USB image and record the printer model, stock version and board markings.
- Identify a non-destructive diagnostic path first: a 3.3 V USB-to-UART
  adapter for console observation, with GND/RX/TX only as verified for the
  exact board; never connect 5 V signalling or VCC from the adapter.
- Establish, per board revision, whether FEL is accessible: exact USB-OTG
  data path, safe entry condition, host enumeration, and a read-only proof
  before any attempt to write storage.  Do not use an ordinary USB-A cable or
  short unidentified pads as an experiment.
- Obtain and verify a complete recovery artifact and a documented target
  storage layout before permitting any FEL write.  Preserve a verified backup
  off-printer; record hashes and a restoration drill.
- If MCU SWD recovery is needed, identify each target and pads electrically,
  confirm 3.3 V logic and shared-ground rules, and use a reputable probe.
  Never power the printer through the probe while its normal power is present.
- Treat every flash operation as an explicit, idle-only, power-stable process
  with a checksum/signature decision, visible progress, bounded failure state
  and a previously tested recovery route.
- Add a release gate: do not describe a project image as installable until its
  normal update/recovery procedure and at least one independently viable
  fallback have evidence for the supported board/version combination.

## Non-goals

- Publishing pinouts, soldering directions, FEL commands or raw storage-image
  write commands before they are physically verified on the applicable AD5M
  board revision.
- Treating a UART console as an eMMC programmer, or treating an ST-LINK as a
  recovery method for the T113 host.
- Replacing the signed project-update/rollback design in AD5M-019.

## Evidence sources

- Local project source: `mcu_update.sh`, the installer and the MCU build
  configurations were inspected on 2026-10-28.
- [Allwinner T113-S3 User Manual, BROM/FEL section](https://raspipcb.com/wp-content/uploads/2025/03/allwinner-t113-user-manual.pdf)
- [sunxi-fel manual](https://github.com/linux-sunxi/sunxi-tools/blob/master/sunxi-fel.1)
- [ST hardware debugger/programmer documentation](https://www.st.com/en/development-tools/hardware-debugger-and-programmer-tools-for-stm32/documentation.html)

## Verification

2026-10-28 source and documentation research only.  No printer, adapter,
probe, FEL command, UART connection, image read/write or MCU operation was
performed.

## Remaining work

1. Inventory the user's available equipment (at minimum, USB-to-UART adapter
   type/voltage, a known-good data cable, multimeter, and any SWD probe).
2. Perform the AD5M-024 physical-identification steps and derive a
   board-revision-specific read-only recovery audit.
3. Create a reviewed recovery procedure and execute a non-destructive drill
   before authorizing any project-image installation.
