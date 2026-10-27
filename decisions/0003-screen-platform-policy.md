# 0003 Retire GuppyScreen; use HelixScreen for future touchscreen work

Status: accepted

## Context

The project needs a touchscreen UI that can be made safe for the AD5M's
resistive display, supported through a reviewable update path, and maintained
without carrying an unnecessary second UI implementation. GuppyScreen added a
separate Buildroot package, local patches, submodules, an overlay, and a
runtime configuration. Its pending security reviews included an unsafe updater
path and unaudited submodule closure.

The project direction is to develop the touchscreen experience with
HelixScreen, keeping it upstream and unmodified where possible. AD5M-005
tracks the first concrete HelixScreen work: an ergonomic layout that separates
easy-to-mistap controls.

## Decision

GuppyScreen is retired. It is not a supported, buildable, or releaseable
variant. Remove its Buildroot package, build target, configuration fragment,
overlay, printer configuration, setup code, and active documentation.

HelixScreen is the sole planned touchscreen UI. This decision does not claim
that HelixScreen has already been packaged or hardware-validated; its version,
integration path, configuration persistence, mock review, and physical-touch
validation remain scoped work in AD5M-005.

Reintroducing GuppyScreen, or adding another touchscreen UI, requires a new
explicit ADR and a fresh supply-chain and safety review.

## Consequences

There is no GuppyScreen build or release path and no need to complete its
updater or submodule-provenance audits. Historical tasks AD5M-010 and AD5M-011
remain rejected records rather than active work. Removing the recipe also
removes its Git and submodule downloads from the build graph.

AD5M-013 retires the remaining KlipperScreen legacy variant, leaving `lite` as
the only supported build variant until HelixScreen is separately integrated and
validated. Any future UI variant requires its own reviewed task.
