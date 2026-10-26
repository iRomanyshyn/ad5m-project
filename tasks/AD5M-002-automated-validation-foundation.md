# AD5M-002 Automated validation foundation

Status: proposed

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

No implementation has started.

## Verification

Not run.

## Hardware verification

Not applicable to the initial software-only foundation.

## Remaining work

Choose the first narrow test set after AD5M-001 defines the canonical build
source relationship.
