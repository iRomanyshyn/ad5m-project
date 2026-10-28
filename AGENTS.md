# AD5M project coordination instructions

This repository coordinates development across the Flashforge AD5M
Klipper Mod project.

## Repositories

Sibling repositories:

- ../ad5m_enhanced_mod
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

### ad5m_enhanced_mod

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

## Capturing ideas

Do not leave actionable ideas, risks, or proposed follow-up work only in chat.
Record them before ending the current work:

- add concise, project-wide ideas to ROADMAP.md;
- create or update a scoped task file for a concrete proposed change;
- create an ADR for a long-lived architectural decision.

Do not turn this into a conversation transcript. Mark unapproved work as
`proposed`, retain known assumptions and limitations, and do not begin it until
the user authorizes that task.

## Durable handover record

The tracked project records must be sufficient for development to resume on a
different workstation or with a different collaborator.  Before ending a work
slice, record each durable outcome in the smallest appropriate place:

- a task for its scope, status, evidence, verification actually performed,
  unresolved risks, and concrete next step;
- ROADMAP.md for a concise cross-project idea not yet scoped; or
- an ADR for a long-lived decision and its alternatives.

Record facts, decisions, links/identifiers, and consequences—not a transcript
of the discussion or speculative narration.  Explicitly distinguish observed
evidence, assumptions, rejected directions, and work that still needs a real
printer.  If an idea was considered but intentionally not retained, say so in
the relevant task/ADR only when that prevents a future unsafe rediscovery.

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
