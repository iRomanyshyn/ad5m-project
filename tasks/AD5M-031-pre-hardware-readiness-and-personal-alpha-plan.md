# AD5M-031 Pre-hardware readiness and personal-alpha plan

Status: in progress

## Goal

Finish the decisions, source reviews, static validation and reproducible
artifacts that do not require an AD5M, then make the first printer sessions
small, observable and reversible.  The target outcome is a controlled
**personal alpha** with documented recovery evidence; it is not a public
firmware release.

## Context

The project has enough source evidence to show that feature work must not
outrun recovery, update and LAN-security controls.  In particular, the legacy
image contains a shared root credential, broad Moonraker trust, automatic MCU
flashing and a destructive USB install path.  AD5M-028 records the source
baseline.  A plan that calls a build successful or an image installable before
these gates is misleading.

This task records the work sequence so it can be resumed on another
workstation without retaining a chat transcript.  It does not authorize a
target write, MCU flash, opening the printer, or a production release.

## Phases and exit criteria

| Phase | Work that can be completed without the printer | Exit criterion |
| --- | --- | --- |
| P0 — source baseline | Complete the privileged-path/source audit, preserve component and binary-input identities, and assign every finding an owner. | AD5M-028 complete; no unowned blocker/high finding. |
| P1 — release/security design | Finish reviewable designs for bootstrap credentials, Moonraker/browser access, general system egress, source/safety patches, media trust, signed update/rollback and release manifest. | Each design names its threat model, immutable inputs, migration/recovery behavior, static tests and hardware evidence still needed.  A design is not an implementation. |
| P2 — software-only engineering | Implement only approved, testable changes: static guards, configuration tests, mock/API tests, package/artifact manifests and reproducible-build evidence. | Clean source validation passes; a release candidate has an exact source/asset inventory and no unresolved static release blocker. |
| P3 — bench readiness | Exercise the user's debug equipment away from the printer: identify cables/adapters, verify 3.3 V UART behavior by loopback, configure capture tools and prepare offline recovery media/checksums. | A written bench record proves the selected console adapter works without powering or touching the AD5M. |
| P4 — read-only printer discovery | With the printer idle, capture stock version/board markings, UART boot observation, actual display/camera/USB topology, storage layout and any safe FEL enumeration evidence. | AD5M-024 and AD5M-026 have board/version-specific evidence; no target storage or MCU was written. |
| P5 — controlled personal alpha | Stage only an artifact that passes P1/P2 gates, use a documented go/no-go checklist and retain an independent recovery route.  Test boot, local UI/API and non-motion service behavior first. | A recorded, bounded alpha result and recovery result for that exact board/stock version.  Motion, heater, probing and print tests remain separately approved hardware work. |

## Mandatory first-install gates

All gates below must be satisfied before calling any project package
installable on the user's printer:

1. No shipped shared root password, packaged private key, universal Moonraker
   trusted-client range or wildcard production CORS (AD5M-016 and AD5M-029).
2. The exact Klipper source, local patch delta and generated MCU artifact
   identities have a reviewed disposition.  A first candidate must not
   automatically flash an MCU merely because it boots, and legacy flash-result
   accounting must not hide a partial failure (AD5M-030, AD5M-034 and
   AD5M-019).
3. The selected installer/update artifact has an authenticated manifest,
   preflight before destructive actions and a recovery route evidenced for the
   actual board/version (AD5M-019, AD5M-025, AD5M-026 and AD5M-027).
4. Every enabled external network path has an explicit policy; local-only UI
   claims exclude hidden upstream update/crash/telemetry transport
   (AD5M-018, AD5M-023 and AD5M-032).
5. A clean build and static validation produce a recorded component/asset
   inventory tied to the exact three-repository revisions.  A dirty local
   `build_output/` directory is not release evidence.
6. The owner has an official compatible stock USB image, a tested console
   capture setup and a written stop condition.  No physical action proceeds
   when the printer is printing, hot, or is the only unverified recovery path.

## Work order

1. Complete P0 (AD5M-028) and maintain this task as the dependency map.
2. Complete P1 source/design work in this order: AD5M-030, AD5M-029 with
   AD5M-016, AD5M-032, AD5M-033, AD5M-034, then AD5M-019/025/027
   reconciliation.
3. Implement P2 changes only after their owning design is accepted.  Prefer
   small static checks and deterministic artifacts over feature additions.
4. Review the P3 checklist with the owner before any cable is connected to an
   AD5M.  P3 may use the user's adapters but is not a printer operation.
5. Start P4 only in a dedicated hardware session.  Stop on an unexpected
   voltage, unlabelled connector, non-stock boot behavior, lost console, or
   any uncertainty about the next write operation.

## Explicit deferrals

- HelixScreen packaging/UI layout, telemetry contribution, mDNS, camera URL
  improvements, swap tuning and optional services may continue only when they
  do not weaken these gates.  They cannot justify a first installation.
- “Public release”, compatibility with unknown stock versions/board revisions
  and unattended updates are out of scope for the personal alpha.
- A successful source build, mocked service test or UART log does not prove
  motion, thermal, probing, storage-write or recovery safety.

## Verification

2026-10-28: created from completed source-level findings and existing task
records.  No source build, target image, installer, adapter or printer was
executed by this planning task.

## Hardware verification

Not run.  P3 onward require a separate, deliberate physical session.
