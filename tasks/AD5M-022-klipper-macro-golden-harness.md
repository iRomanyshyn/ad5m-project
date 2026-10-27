# AD5M-022 Klipper macro golden-test harness

Status: complete

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

## Implementation notes

### 2026-10-27 first official-Klipper harness

The first implementation is `make macro-golden` in the integration tree.  It
requires an explicit `KLIPPER_SOURCE_DIR`; it does not clone, download, use a
submodule, or read ignored `build_output/`.  The fixture accepts only a clean
checkout whose `origin` is exactly `https://github.com/Klipper3d/klipper.git`,
whose `HEAD` is official `v0.13.0` commit
`61c0c8d2ef40340781835dd53fb04cc7a454e37a`, and whose
`klippy/extras/gcode_macro.py` SHA-256 is
`4e3ebcda1bd9facb7e9805dcf37a37d943a1979b9e994050cd16ae67188c2401`.
The release identifier is a lightweight tag, so it has no tag signature.  A
temporary official clone passed `git fsck --no-dangling`.

The runner imports only that upstream module and supplies a minimal fake
printer/config registry necessary to construct `PrinterGCodeMacro` and call
its `TemplateWrapper.render()`.  It intentionally never constructs a macro
command executor or any serial, CAN, MCU, Moonraker, camera, network, or
hardware object.  It also sets `sys.dont_write_bytecode`, leaving the supplied
checkout untouched.

`_LOAD_MATERIAL_ACTION` rendered nine exact `RESPOND` prompt commands matching
the tracked golden file.  Before comparison, every rendered command is
restricted to the fixture's `RESPOND` allow-list; a tracked fixture containing
`G1 X1 F600` proves a physical command is rejected.  This demonstrates
template/control-flow rendering only.  It neither executes button targets nor
validates UI rendering, touch, filament, temperatures, movement, or any
printer behavior.

To reproduce with a separately obtained official source checkout:

```sh
git -C /path/to/klipper checkout --detach 61c0c8d2ef40340781835dd53fb04cc7a454e37a
make -C ad5m_enhanced_mod macro-golden KLIPPER_SOURCE_DIR=/path/to/klipper
```

## Acceptance criteria

- A clean host can run the fixture against the recorded Klipper revision.
- The fixture fails clearly if the prompt sequence or an output field changes.
- A negative fixture proves that an unexpected physical or external command is
  rejected rather than silently accepted.
- No test result is described as hardware, UI touch, filament, thermal, or
  printer safety verification.

## Verification

- `git ls-remote https://github.com/Klipper3d/klipper.git refs/tags/v0.13.0`
  resolved the official lightweight tag to
  `61c0c8d2ef40340781835dd53fb04cc7a454e37a`.
- A temporary official-only shallow checkout at that exact commit passed
  `git fsck --no-dangling`; no source from `build_output/` was executed.
- `make -C ad5m_enhanced_mod macro-golden
  KLIPPER_SOURCE_DIR=<official-clean-checkout>` — passed twice, rendering nine
  `RESPOND` commands and leaving the supplied source checkout unchanged.
- The tracked `unexpected-command.gcode` fixture, containing `G1 X1 F600`, was
  rejected before comparison as an unexpected command.
- A temporary copy with an intentionally changed prompt label failed with the
  explicit `golden output mismatch` diagnostic.

These are software-only template/control-flow results.  They do not execute
the prompt-button target macros and do not establish UI, touch, filament,
thermal, motion, MCU, network, or printer behavior.

## Hardware verification

Not applicable to the harness itself.  Any macro change affecting printer
behavior retains its own explicit hardware-validation requirement.
