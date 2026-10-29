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

### Available-equipment inventory (user report, 2026-10-28)

| Equipment | Best AD5M role | Status / constraint |
| --- | --- | --- |
| Waveshare FTDI USB-to-TTL | Primary host UART console. | Appropriate after its I/O level is explicitly set to 3.3 V. Connect only GND, crossed TX/RX; leave VCC disconnected. The cited Waveshare FT232 variant supports a 3.3/5 V switch, but the setting must be checked on the actual unit. |
| WeAct Studio Mini Debugger (ST-LINK/V2.1) | Candidate SWD probe for a physically verified STM32 target. | Its project documents ST-Link/DAPLink firmware. Do not assume its exposed connector is a ready USB-UART until a bench loopback proves the installed firmware and pins. |
| Waveshare RP2350-GEEK | Backup USB-UART and CMSIS-DAP/SWD probe. | Waveshare documents 3-pin UART and CMSIS-DAP SWD, selected by firmware. Load/identify the Debug Probe firmware and perform a loopback before relying on it. |
| Raspberry Pi 4 / 3B | Spare 3.3 V UART-console host or development machine. | Suitable with a configured GPIO UART, but less convenient than the FTDI for first diagnosis. GPIO is not 5 V tolerant. |
| Android with USB-C OTG | Emergency terminal host for a known-working USB-UART adapter. | Convenience fallback only; driver/application behavior must not be the sole recovery plan. |
| Other RP2350/ESP boards | Custom tooling reserve. | Out of scope until the basic FTDI UART and recovery path are proven. |

No dedicated SPI/eMMC programmer or board-specific test clip is currently in
the reported kit.  That is not yet a purchase requirement: first identify the
actual storage, recovery interfaces and failure modes.  A generic clip or
programmer cannot safely be selected without that evidence.

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
- [Waveshare FT232 USB-to-TTL specifications](https://www.waveshare.com/usb-to-ttl-c.htm)
- [Waveshare RP2350-GEEK documentation](https://www.waveshare.com/wiki/RP2350-GEEK)
- [WeAct Mini Debugger repository](https://github.com/WeActStudio/WeActStudio.MiniDebugger)
- [Raspberry Pi UART documentation](https://www.raspberrypi.com/documentation/computers/configuration.html)

## Verification

2026-10-28 source and documentation research only.  No printer, adapter,
probe, FEL command, UART connection, image read/write or MCU operation was
performed.

### 2026-10-29 bench UART-loopback aid

`ad5m_enhanced_mod/tools/uart-loopback-check.sh` provides a deliberately
limited host-side check for a USB-UART adapter. It accepts one explicit
`/dev/tty*` character-device path and refuses to write unless the caller sets
`AD5M_UART_LOOPBACK_CONFIRM=TX_RX_SHORTED`. With the adapter disconnected
from all targets, configured for 3.3 V where applicable, and only its TX/RX
pins shorted together, it sends a nonce at 115200 8N1 and requires the same
line back. It restores the prior serial settings on exit.

Example, after identifying the actual adapter path:

```sh
cd ad5m_enhanced_mod
AD5M_UART_LOOPBACK_CONFIRM=TX_RX_SHORTED \
  ./tools/uart-loopback-check.sh /dev/ttyUSB0
```

A pass establishes only host adapter, permissions, TX/RX loop and the chosen
bench serial setting. It does not establish an AD5M connector, pinout, voltage
level, boot log, UART speed, FEL availability or recovery capability. The tool
was syntax-checked only; no adapter or printer was connected in this work.

### P3 bench-record template

Record this once per chosen adapter, with the adapter disconnected from every
printer, SBC and debugger target. Store the completed record in a local,
ignored note or attach its non-secret facts to this task; do not store terminal
logs that may later contain credentials.

```text
date/time and workstation:
adapter make/model and visible USB identifier:
selected adapter voltage marking/switch position: 3.3 V / verified how:
host device path used:
only physical connection during test: TX shorted to RX; VCC disconnected
command/version of uart-loopback-check.sh:
result at 115200 8N1: pass / fail
host permissions or driver issue, if any:
```

An incomplete or failed record means only that the adapter is not ready for a
printer session. A successful record is not permission to choose an AD5M
header, attach SWD/FEL, power a target, read a boot log or change storage. The
first printer session must still use the read-only AD5M-024/031 entry record,
verified connector evidence and its stop conditions.

## Remaining work

1. Inventory the user's available equipment (at minimum, USB-to-UART adapter
   type/voltage, a known-good data cable, multimeter, and any SWD probe).
   Initial inventory is recorded above; bench-test FTDI at 3.3 V and the
   selected RP2350/WeAct UART capability with a loopback, without the printer.
2. Perform the AD5M-024 physical-identification steps and derive a
   board-revision-specific read-only recovery audit.
3. Create a reviewed recovery procedure and execute a non-destructive drill
   before authorizing any project-image installation.
