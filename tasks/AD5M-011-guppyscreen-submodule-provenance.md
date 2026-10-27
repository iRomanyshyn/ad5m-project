# AD5M-011 GuppyScreen submodule provenance

Status: proposed

## Goal

Audit the complete source closure fetched by the GuppyScreen Buildroot recipe
before the GuppyScreen variant is treated as release-ready.

## Context

The recipe pins GuppyScreen itself to
`fdca5728001f7a6f6ce774dcfa3998f8b96e6878` and enables
`GUPPYSCREEN_GIT_SUBMODULES = yes`. That commit fixes Gitlink IDs for
`libhv`, `lv_drivers`, `lvgl`, and `spdlog`, but Buildroot will still fetch
their source through URLs recorded in `.gitmodules`. AD5M-008 audited the
parent source and its direct packaged binary only; it did not independently
audit each fetched submodule's provenance, license, revision, or build input.

## Requirements

- Record every Gitlink path, commit ID, remote URL, license, and relevant
  transitive-submodule state for the exact parent commit.
- Verify each source object with `git fsck`; determine whether signed tags or
  commits are available, without treating an unsigned tag as authenticated.
- Inspect the build recipe's local patches and the source areas they affect.
- Confirm that all network downloads during a GuppyScreen build correspond to
  the recorded closure, and retain no mutable branch dependency.
- Run a separate cross-build after review and record its result. Do not install
  it on a printer as part of this task.
- Keep untrusted historical sources out of the audit inputs and implementation.

## Non-goals

- Enabling the GuppyScreen runtime self-updater; that is AD5M-010.
- Treating the parent Git commit alone as a substitute for submodule review.

## Verification

Not run.

## Hardware verification

Not run.
