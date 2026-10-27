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

### 2026-10-27 controlled Buildroot configuration check

`make validate-config` now runs the separate tracked
`tests/validate-buildroot-config.sh` entry point.  It resolves the canonical
Buildroot checkout, requires its `HEAD` to equal the 40-character revision in
`build_scripts/buildroot/buildroot-revision`, and rejects a dirty checkout
before Buildroot is invoked.  It then creates a unique directory beneath
`TMPDIR` and calls only `prepare_variant lite` with `AD5M_BUILD_OUT` set to
that directory.  The normal build output remains
`ad5m_enhanced_mod/build_output`; the override exists solely so disposable
validation can avoid it.

The check asserts that Buildroot produced the `lite` `.config`, that the
generated configuration has the project-owned common and lite rootfs overlays,
and that the expected Moonraker and ustreamer selections remain enabled.  The
temporary output is removed by a shell trap on success, failure, or
interruption.  This is intentionally a configuration integration check, not a
target image build: Buildroot may compile its small host-side Kconfig helper,
but it does not build target packages, download package sources, execute
target code, or establish any printer behavior.

### 2026-10-27 portable host lint baseline

`make lint` runs the tracked `tests/lint-host.sh` without a third-party host
formatter dependency.  It compiles every active Python source under the
integration build inputs with `python3 -m py_compile`, setting
`PYTHONPYCACHEPREFIX` to a temporary directory so no `__pycache__` is written
to the repository.  It also enforces no trailing whitespace and a final
newline in project-owned test shell/Python sources.  `make validate` now runs
this lint baseline before the existing static integration checks.

This is deliberately a conservative starting gate.  A diagnostic full Ruff
scan of the inherited runtime modules reports 253 existing findings, many in
the safety-adjacent `timelapse.py`; neither auto-formatting nor broad
behaviour-changing fixes are appropriate in a validation-foundation change.
That remediation is tracked separately as AD5M-021.

## Verification

- `bash -n ad5m_enhanced_mod/tests/validate-static.sh`
- `make -C ad5m_enhanced_mod validate` — passed: sole `lite` variant, both
  retired targets unavailable, negative fixture rejected, no submodule or
  legacy-reference finding, valid Bash syntax for the top-level wrapper, and
  syntax passed for 32 active shell scripts in total.
- `tests/validate-static.sh ad5m_enhanced_mod/build_output` — expected failure
  confirmed; the runner rejects a generated-output directory as its root.
- `bash -n ad5m_enhanced_mod/build_scripts/env.sh`
- `bash -n ad5m_enhanced_mod/tests/validate-buildroot-config.sh`
- `make -C ad5m_enhanced_mod validate-config` — passed against canonical
  Buildroot `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de`; initialized and removed
  only a unique temporary Buildroot output directory, and verified the
  generated lite configuration's overlays, Moonraker, and ustreamer.
- `make -C ad5m_enhanced_mod lint` — passed: compiled four active Python
  files with disposable bytecode and checked whitespace hygiene in three
  project-owned test sources.
- `make -C ad5m_enhanced_mod validate` — passed with both the lint and static
  integration layers.
- `python3 -m compileall` with a temporary `PYTHONPYCACHEPREFIX` — passed for
  all four active integration Python files.
- `ruff check --select E9,F63,F7,F82` — passed for those same files as a
  diagnostic sanity check; full Ruff findings are intentionally deferred to
  AD5M-021 rather than auto-fixed.

These are software-only checks.  The static runner did not configure
Buildroot; the separate configuration runner did configure it in temporary
output but did not compile a target image, execute target code, or establish
hardware behavior.

## Hardware verification

Not applicable to the initial software-only foundation.

## Remaining work

1. Define Golden G-code fixtures only for a reviewed, non-hardware-emulating
   macro behavior with a clear expected output.
