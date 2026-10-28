# AD5M-021 Runtime Python lint remediation

Status: in progress

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

2026-10-28 software-only checks passed:

- `make -C ad5m_enhanced_mod lint`
- `make -C ad5m_enhanced_mod lint-ruff`

The Ruff command uses an ephemeral cache and checks no target artifacts.  It
does not run a Klipper, Moonraker, camera, FFmpeg or printer workflow.

## Implementation notes

### 2026-10-28 narrow host-only initial gate

The active runtime Python surface is four GPLv3-headered modules.  Their
headers identify `gcode_shell_command.py` with Eric Callahan, `md5_checker.py`
with minicx, `power_loss_recovery.py` with Ankur Verma, and `timelapse.py`
with Christoph Frei.  The local history shows that the shell-command and
timelapse modules entered through the inherited integration history, while
the MD5 checker and power-loss recovery have single historical introduction
commits.  This is copyright/arrival evidence, not a completed source-revision
closure audit; before behavior changes each module still needs a separately
recorded upstream or author-source identity.

Ruff `0.17.0` is now pinned in `requirements-dev.txt` and required by
`ruff.toml`; the target baseline is the pinned Buildroot Python 3.12 series.
`make lint-ruff` is deliberately opt-in and never installs a package, writes
inside the checkout, or becomes a target-image dependency.  The existing
portable `make lint` and `make validate` remain usable on a host without Ruff.

The initial policy selects only `F401` (unused imports) for exactly the four
runtime modules.  That is intentionally smaller than the 253-finding
diagnostic full scan: it excludes formatting, exception handling, logging,
typing modernization and suggested code transformations until focused tests
cover each behavior-sensitive module.  The sole current finding was an unused
type-checking-only `WebCam` import in the timelapse extension; removing that
name cannot change runtime imports or the timelapse command path.

The next increment must be selected rule-by-rule with per-module tests and a
documented reason.  In particular, do not enable broad `F`/`E`/`B`/`S`/`UP`
families or run `ruff --fix` across power-loss recovery or timelapse merely to
reduce the count.
