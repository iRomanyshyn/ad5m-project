# AD5M-037 K1/K1C ecosystem idea triage

Status: triaged — no feature adoption

## Purpose

Preserve a user-supplied list of architectural ideas prompted by the Creality
K1/K1C mod ecosystem, and compare it to the AD5M backlog. This is a planning
cross-check only: it is not a source review of another project, an endorsement
of that ecosystem, an implementation instruction, or authority to import any
code.

The AD5M remains a constrained, AD5M-specific product. Its existing load-cell
probing system and recovery constraints take priority over generic Klipper
platform breadth.

## Overlap and gaps

| Idea | Existing AD5M coverage | Gap or disposition |
| --- | --- | --- |
| Componentized install/removal | AD5M-009 defers media to a separately reviewed add-on; AD5M-015/018/023 own HelixScreen packaging and update boundaries; AD5M-019 defines project-managed release/update authority; AD5M-036 scopes a future media job. | **Partial.** There is no accepted component lifecycle contract covering compatibility with Core, permissions, disk/RAM budget, installation/removal, persistent-state ownership, rollback, and support status. Do not add generic plug-in machinery before that contract exists. |
| Backup/restore before updates | AD5M-019 inventories persistent state, migrations, rollback compatibility and release manifests; AD5M-026 owns recovery evidence. | **Partial.** Project-internal migration is not a user-facing backup/restore feature. A later task must define an explicit, secret-aware export/import format, supported state, encryption/consent, compatibility, integrity and recovery semantics. |
| Independent management of non-critical services | Existing init scripts and AD5M-004/015 touch individual services. | **Missing.** Define a small service-control contract for camera, Moonraker and a future UI: health indication, restart rate limit, no printer reboot by default, no restart of Klipper/MCU paths without a separate safety decision, and user-visible failure state. |
| Camera recovery/configuration | AD5M-004 owns stable UI/camera access and camera audience; AD5M-024 owns physical USB/camera topology. | **Partial.** Automatic registration, safe restart after unplug/replug, persistence/override policy, and no-camera behaviour need an explicit camera-lifecycle task after topology evidence. |
| Keep stock load-cell probing; do not pursue generic alternative probes | Load-cell is explicitly safety-critical in `PROJECT.md`; AD5M-030 audits forked load-cell behavior; AD5M-035 addresses a concrete diagnostic memory-safety defect. | **Covered direction.** Do not add Eddy, Cartographer, Beacon or other probe support absent a real AD5M requirement. This is a scope boundary, not a claim that such hardware is unsuitable in general. |
| Installer/generated-config/migration/variant tests | AD5M-002 is the validation foundation; AD5M-003 scopes mocks; AD5M-022 covers macro goldens; AD5M-027 checks installer ordering. | **Partial.** Expand only with deterministic fixtures: generated config, state migration/rollback, service policy and 5M/5M Pro compatibility fixtures. Never substitute a project-written printer emulator for hardware evidence. |
| Resonance, belt and controlled-axis diagnostics | General diagnostics appear in `PROJECT.md`; AD5M-006 covers resource pressure. | **Missing.** A future optional diagnostics profile needs exact sensor/tool provenance, storage/CPU limits, controlled excitation safeguards, result format and hardware test plan. It must not be folded into Core by default. |
| Adaptive/full/existing/off bed-mesh policy via native Klipper | `PROJECT.md` already selects native adaptive mesh and rejects adding KAMP just for duplicate behavior. | **Partial.** The product policy and UX/macro test matrix do not yet have a dedicated owner. Add one only when normal AD5M probing/load-cell hardware evidence makes the policy testable. |
| Recovery/unbrick deliverable for T113-S3 | AD5M-026 owns board-specific recovery/reflashing readiness; AD5M-024 owns topology; AD5M-031 makes recovery a personal-alpha gate. | **Covered.** No unverified FEL/SWD/storage-write procedure may be inferred from a different printer family. |
| Official-Buildroot-based userspace | AD5M-001/002 establish the canonical official-Buildroot-derived source contract; AD5M-007/008 cover package/build provenance. | **Covered and already adopted.** |

## Core versus optional split

The suggested split is directionally compatible with `PROJECT.md`'s rule that
Klipper/MCU communication takes priority over optional services. It is not yet
an accepted packaging matrix.

- **Candidate Core:** Klipper, Moonraker, verified AD5M load-cell support,
  networking, recovery/update controls, and only minimal diagnostics needed to
  operate/recover the printer.
- **Candidate optional components:** HelixScreen, camera lifecycle features,
  media/timelapse, advanced diagnostics, extra macros, memory tuning and
  remote integrations.

Two product decisions remain open and must not be silently inferred from the
table: whether a responsive on-printer UI is required in the first personal
alpha, and whether basic camera support is baseline or optional. The current
base is headless; HelixScreen is the sole planned UI but has not passed source
closure or hardware validation. The current camera stack exists, but its
topology, resource cost and lifecycle are not yet validated.

## Priority conclusion

Reliability, recovery and maintainability of the existing AD5M take priority
over platform breadth. The next work should continue to close AD5M-019/024/026
and the existing validation/recovery gates; the missing ideas above are
backlog candidates, not reasons to delay or expand the first hardware
discovery session.
