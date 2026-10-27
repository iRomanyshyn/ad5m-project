# 0004 Source hierarchy and research policy

Status: accepted

## Context

AD5M work needs to learn from the stock printer and from community experience
without turning privileged firmware, installer, or update code into an
unreviewed supply-chain dependency. The project has official upstreams, an
official vendor-published AD5M Klipper repository, historical references, and
independent community mods with different maintenance and security practices.

## Decision

Use the following hierarchy:

1. Official upstream projects — Klipper, Moonraker, Buildroot, and
   HelixScreen — are the preferred source for supported functionality.
2. Flashforge's published `FlashForge/AD5M_Series_Klipper` repository is a
   vendor reference for AD5M-specific behavior. Before any use, verify the
   repository identity, pin an exact revision, respect its license, and review
   only the files relevant to a defined question.
3. Historical and community projects, including Z-mod and other AD5M mods,
   are untrusted research inputs. They may be inspected for ideas and observed
   behavior, but are never authoritative upstreams or automatic dependencies.

No external source may be merged, rebased onto, bulk cherry-picked, copied as
a tree, or introduced as a build dependency. A candidate change must first be
identified precisely, compared with official upstream support, independently
implemented in the smallest practical form, and reviewed for security,
licensing, safety, update behavior, and resource cost.

The untrusted-input designation is technical and applies regardless of an
author's nationality or identity. It is a defensive default for privileged
embedded software; it must not be presented as evidence that a particular
project contains a backdoor or malicious code without concrete evidence.

## Consequences

The project may benefit from vendor and community knowledge while retaining an
auditable source and update graph. Research findings belong in a scoped task;
approved implementation and hardware validation remain separate work. AD5M-014
tracks the first vendor-reference audit.
