# 0002 Canonical Buildroot source contract

Status: accepted

## Context

The Klipper Mod build wrapper formerly selected a historical third-party
Buildroot git submodule. That source was neither the project's authoritative
upstream nor reproducible from the canonical sibling repositories.

## Decision

The Klipper Mod external tree uses the sibling
`../buildroot-flashforge-ad5m` checkout by default. The exact allowed
Buildroot commit is tracked in
`ad5m_enhanced_mod/build_scripts/buildroot/buildroot-revision`; the wrapper
checks that revision and rejects a dirty Buildroot source before configuration.

The project does not use a Buildroot git submodule. Buildroot output configured
from another source is not reused automatically. AD5M-specific packages and
configuration remain in `ad5m_enhanced_mod`; generic changes remain in the
official-Buildroot-derived base repository.

## Consequences

Changing the Buildroot revision is an explicit cross-repository change. It
requires updating the revision contract, recording the compatible commit pair
and validation in the scoped task, and rebuilding local output as necessary.
This adds a small maintenance step but prevents silent supply-chain changes and
mixed Buildroot output.
