# AD5M-030 Klipper source and safety-patch audit

Status: proposed

## Goal

Establish a reviewable provenance and safety delta for the pinned Klipper
source and each local patch before its host code or generated MCU firmware is
treated as releaseable.

## Context

The project package currently retrieves commit
`fc2aaae692b5f703420915390a3dcb8cd514c384` from `xblax/klipper`.  It also
applies local patches, including a TRSYNC timeout relaxation, and builds
mainboard/eboard firmware from configured targets.  A Git commit pin is not a
substitute for reviewing the fork's divergence from official Klipper,
Flashforge-specific behavior, patch provenance or physical safety impact.

This is not an allegation about the fork or its authors.  It is a required
control for any privileged, safety-relevant source input under ADR-0001 and
ADR-0004.

## Requirements

- Identify the exact relation of the pinned commit to official Klipper and
  record a minimal, classified delta.  Do not merge, rebase or bulk-import
  history as an audit shortcut.
- Attribute every project-local patch, state its intended behavior and review
  its changed code paths.  For timing, probing, load-cell, heaters, motion,
  serial transport, MCU build/flash or power behavior, require a separate
  safety argument and hardware-validation plan.
- Compare AD5M-specific behavior against the vendor reference only for named,
  minimal questions at pinned revisions; do not treat vendor source as an
  automatic replacement.
- Produce a reproducible source/component record with commit IDs, licenses,
  patches and generated MCU artifact identities.
- Do not swap the source, relax/restore a timeout, or flash an MCU as part of
  this audit.  Any change needs an approved implementation task.

## Verification

Not run.  AD5M-028 records only static package evidence.

## Hardware verification

Not run.
