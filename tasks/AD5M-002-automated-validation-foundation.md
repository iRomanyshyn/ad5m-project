# AD5M-002 Automated validation foundation

Status: in progress

## Goal

Create a fast, layered software-validation baseline that catches regressions
without requiring access to a physical printer.

## Context

The project needs repeatable checks before firmware or safety-critical changes
reach hardware. Klipper can support regression and batch G-code comparison;
Buildroot provides static/package checks and generic runtime-test facilities.
These checks complement, but cannot replace, printer testing.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- Start with checks that are deterministic, non-destructive, and feasible on a
  Fedora development workstation or CI worker.
- Include formatting/lint checks appropriate to changed shell and Python code.
- Validate relevant Buildroot configuration and package metadata.
- Add Klipper regression or batch G-code golden tests for AD5M macros where
  practical.
- Store test fixtures and expected outputs in Git; keep build output local.
- Clearly label every result as software-only verification.

## Non-goals

- Claiming motion, heater, probe, load-cell, MCU, or printing safety.
- Building a full AD5M hardware emulator.
- Requiring the historical third-party Buildroot source.

## Proposed implementation

Define a minimal CI entry point, add static checks first, then introduce small
golden G-code scenarios that compare intentional Klipper command output. Add
only tests with a clear owner and failure diagnosis.

## Acceptance criteria

- A documented command executes the selected checks from a clean checkout.
- A deliberately invalid fixture causes a clear failure.
- Test results distinguish configuration/build checks from hardware evidence.
- The checks do not initialize the legacy Buildroot submodule.

## Implementation notes

### 2026-10-27 first static-test boundary

AD5M-001's canonical Buildroot relationship is now defined, but the external
tree has no `tests/`, CI workflow, `validate` Make target or host-side test
runner.  Its top-level Makefile currently declares `VARIANTS = lite`; the
observed `make help` output lists only that variant.  Dry-running the removed
`guppyscreen` and `klipperscreen` targets fails with "No rule to make target",
and the active source scan has no reference to either legacy UI outside local
generated output.

The first implementation should remain deliberately small: add one
project-owned Bash static-test entry point (for example
`tests/validate-static.sh`) and a `make validate` wrapper.  It must take the
external-tree root as its only input, create no build output, and exercise
only ordinary Fedora host tools (`bash`, `sh`, `make`, `find`, `grep`,
`git` and `mktemp`).  Its initial assertions should be:

1. `make help` exposes `lite` as the sole supported variant.
2. `make -n guppyscreen` and `make -n klipperscreen` both fail; a test fixture
   with a deliberately reintroduced legacy target must prove the check fails.
3. Active Makefiles, Buildroot package/config registration, overlays,
   post-build logic and supported documentation contain neither legacy UI.
   Historical task records and ignored `build_output/` are explicitly outside
   that production-source scope.
4. Shell syntax checks cover the top-level build wrapper and every project
   shell file that the static package path invokes, including `post_build.sh`.
5. The test rejects an accidental build-output path or legacy Buildroot
   submodule reference rather than initializing, fetching or compiling it.

The runner should print each failed invariant and the inspected path, not a
generic pass/fail result.  `make lite_prepare` remains a separately named
configuration-integration check because it can initialise Buildroot state;
neither it nor a full image build belongs in the first static target.

### 2026-10-27 initial implementation

The external tree now provides `make validate`, which runs the tracked
`tests/validate-static.sh` entry point.  It rejects a root path inside
`build_output/`, does not call the build wrapper, and creates only a temporary
host directory for captured diagnostics.  The initial checks cover the single
`lite` variant, the expected failure of both retired screen targets, absence
of submodule metadata and active legacy-source references, and syntax of the
active shell scripts below `build_scripts`, `build_overlays` and
`device_files`.  The tracked `tests/fixtures/legacy-ui/Makefile` deliberately
declares `lite guppyscreen`; the same variant parser must reject it for the
test to pass.  The implementation also corrected `build_scripts/build.sh` to
use its required `#!/bin/bash` shebang and validates that wrapper explicitly;
it uses Bash syntax and must not depend on an incidental `/bin/sh` choice.

## Verification

- `bash -n ad5m_enhanced_mod/tests/validate-static.sh`
- `make -C ad5m_enhanced_mod validate` — passed: sole `lite` variant, both
  retired targets unavailable, negative fixture rejected, no submodule or
  legacy-reference finding, valid Bash syntax for the top-level wrapper, and
  syntax passed for 32 active shell scripts in total.
- `tests/validate-static.sh ad5m_enhanced_mod/build_output` — expected failure
  confirmed; the runner rejects a generated-output directory as its root.

These are software-only static checks.  They did not configure Buildroot,
compile a target image, execute target code, or establish hardware behavior.

## Hardware verification

Not applicable to the initial software-only foundation.

## Remaining work

1. Add the separately controlled canonical-Buildroot configuration check.
2. Add lint/format checks appropriate to newly introduced shell and Python
   files, without making unavailable host tooling a target-image dependency.
3. Define Golden G-code fixtures only for a reviewed, non-hardware-emulating
   macro behavior with a clear expected output.
