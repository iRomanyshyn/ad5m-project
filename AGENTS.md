# AD5M project coordination instructions

This repository coordinates development across the Flashforge AD5M
Klipper Mod project.

## Repositories

Sibling repositories:

- ../flashforge_ad5m_klipper_mod
- ../buildroot-flashforge-ad5m

This repository contains project-wide context, tasks and architectural
decisions. It does not contain firmware source code itself.

## Before working on a task

1. Read PROJECT.md.
2. Read the active task under tasks/.
3. Read AGENTS.md in every repository affected by the task.
4. Inspect the current branch and git status of every affected repository.
5. Inspect both code repositories when ownership of a change is unclear.

Do not assume that a workaround belongs in Klipper Mod if the underlying
problem belongs in Buildroot.

Do not assume that a generic Buildroot modification belongs in the base
Buildroot tree if it can cleanly live in the Klipper Mod external tree.

## Repository responsibilities

### flashforge_ad5m_klipper_mod

Owns project-specific integration, including:

- printer configuration
- Klipper macros
- Moonraker configuration
- printer services
- installer/update logic
- firmware packaging
- UI integration
- Buildroot external-tree configuration and project-specific packages

### buildroot-flashforge-ad5m

Owns changes that properly belong to the Buildroot base, including:

- generic Buildroot fixes
- reusable package fixes
- missing generic dependencies
- base userspace
- toolchain/base-system changes

Its primary upstream is the official Buildroot project.

## Project records

PROJECT.md contains stable project-wide architecture and policy.

ROADMAP.md contains the high-level backlog.

tasks/ contains actionable scoped work items.

decisions/ contains architectural decisions that should survive individual
tasks.

Task-specific discoveries belong in the task file rather than PROJECT.md.

## Git behavior

Do not commit, push, merge, rebase, force-push or open pull requests unless
the user explicitly requests it.

Never claim that a build or hardware test was performed unless it was
actually performed.

When completing work, report:

- repositories affected
- files changed
- tests/builds actually run
- hardware verification still required
- unresolved risks or follow-up work
