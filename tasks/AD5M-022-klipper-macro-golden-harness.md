# AD5M-022 Klipper macro golden-test harness

Status: proposed

## Goal

Add a reproducible, isolated software-only way to render selected AD5M
Klipper macros and compare their intentional G-code or UI-response output to
tracked golden fixtures.

## Context

AD5M-002 already provides static, host-lint, and generated-configuration
checks.  It must not claim that a handwritten Jinja parser, a direct string
comparison, or a physical-printer configuration parser validates a macro's
runtime behaviour.

The current macro inventory is dominated by motion, heater, fan, display,
filament, shutdown, and MCU-affecting behavior.  The first viable fixture
candidate is `_LOAD_MATERIAL_ACTION` in `printer_configs/macros.cfg`: when
rendered, it produces only Moonraker/Fluidd `RESPOND` prompt commands.  It
does not itself move an axis, heat an extruder, toggle a fan, execute a shell
command, or invoke the load/unload/purge actions selected later by a user.

## Requirements

- Use a pinned, reviewed Klipper source/harness; do not execute a source tree
  recovered from ignored `build_output/` or import code from a historical
  third-party repository merely to run tests.
- Run in a disposable host environment with no real serial, CAN, Moonraker,
  camera, network service, or printer endpoint.
- Provide minimal fake objects only at documented Klipper extension points and
  fail if an unexpected command would cause motion, heating, MCU traffic,
  shell execution, or a network action.
- Store the macro input, exact expected prompt-command output, source revision,
  and command to run it in Git.
- Explicitly label every result as template/control-flow coverage only.

## Non-goals

- Emulating a printer, temperatures, motion, load cell, MCU firmware, or
  Moonraker service.
- Testing `START_PRINT`, leveling, filament movement, fans, power, shutdown,
  screen backlight, or any macro that controls physical equipment.
- Using test output as approval to install a build on hardware.

## Candidate first fixture

Invoke `_LOAD_MATERIAL_ACTION` with no parameters and assert the ordered
`action:prompt_*` output for the title, text, Load/Unload/Purge buttons, Done
button, and show command.  The fixture must assert command ordering and exact
labels, but the harness must block execution of the button targets; their
physical effects are outside this test.

## Acceptance criteria

- A clean host can run the fixture against the recorded Klipper revision.
- The fixture fails clearly if the prompt sequence or an output field changes.
- A negative fixture proves that an unexpected physical or external command is
  rejected rather than silently accepted.
- No test result is described as hardware, UI touch, filament, thermal, or
  printer safety verification.

## Hardware verification

Not applicable to the harness itself.  Any macro change affecting printer
behavior retains its own explicit hardware-validation requirement.
