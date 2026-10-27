# AD5M-021 Runtime Python lint remediation

Status: proposed

## Goal

Establish a reviewed, meaningful Python lint policy for the AD5M-owned runtime
extensions without applying broad automated rewrites to printer-related code.

## Context

AD5M-002's portable lint baseline compiles all four active Python sources and
keeps bytecode out of the repository.  On 2026-10-27, a diagnostic full Ruff
scan reported 253 existing findings in the inherited extension sources, with a
large concentration in `build_scripts/components/moonraker_extensions/timelapse.py`.
The results include style findings, catch-all exception handling, logging
patterns, and suggested transformations that could change runtime behaviour.

## Scope

- Classify every active Python file as project-authored, upstream-derived, or
  third-party-derived, with source and license provenance.
- Select an explicit Python target version and a version-pinned lint tool for
  development/CI only.
- Triage diagnostics into safe mechanical formatting, correctness defects, and
  intentional integration behaviour.
- Add narrowly scoped tests before changing timelapse, print recovery,
  checksum, or shell-command behaviour.
- Introduce the lint gate incrementally with documented, reviewed exclusions
  where an exception remains intentional.

## Non-goals

- Run `ruff --fix`, including unsafe fixes, over the inherited tree.
- Treat lint-clean output as proof of print, motion, thermal, or camera
  behaviour.
- Add a target-image runtime dependency solely for host-side linting.

## Acceptance criteria

- The lint configuration and tool version are reproducible in development and
  CI.
- Every exception is local, justified, and has an owner or removal condition.
- Behaviour-sensitive changes have focused software tests and clearly marked
  hardware verification requirements.
- The default AD5M-002 checks remain usable on a Fedora host without silently
  downloading or modifying target artifacts.

## Verification

Software-only until focused runtime tests and any required hardware checks are
recorded with individual changes.
