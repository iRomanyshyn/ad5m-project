# AD5M-028 Phase-zero provenance and privileged-path audit

Status: complete

## Goal

Create a concise, evidence-backed baseline of the release input graph and
privileged execution paths before accepting further release-facing features or
describing an image as distributable.

## Context

The project already has focused provenance and safety records for selected
components (including package closure, runtime Python sources, display
backlight, recovery and removable media).  They are not a substitute for one
cross-cutting report that a later maintainer can use to find every remaining
high-risk source, installer, init/service entry point, network boundary and
safety-sensitive patch.

This task is a source audit only.  It must not infer an author's nationality
or intent from a name, repository or origin; no finding is evidence of a
backdoor without concrete behavior.  Historical/community repositories remain
untrusted research inputs under ADR-0001 and ADR-0004 regardless of identity.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m, read-only unless a later, approved generic fix is
  established

## Audit inventory

Record, with exact paths and revisions/hashes where available:

1. Every project-specific fetched source, prebuilt/archive asset and locally
   carried executable component; identify source URL, retrieval method, pin or
   digest, license record and whether a focused provenance review exists.
2. Every host-side and chroot-side root execution entry point: installer,
   host init, init service, mdev handler, update/recovery hook and network
   daemon.  Distinguish boot-time execution from a user-requested operation.
3. Declared or hard-coded network endpoints, listeners and outbound-capable
   code paths.  Distinguish loopback/LAN listeners from external egress and
   from a string that is documentation only.
4. Patches or local components touching load-cell/probing, heaters, motion,
   MCU flashing, power control or network authorization.  Compare to official
   upstream/vendor references only for a precisely scoped question; do not
   bulk import or rebase.
5. Legacy release/documentation claims, shared credentials and source links
   that conflict with current project policy.

## Deliverable and triage

Produce a ranked findings table with source evidence, current control,
limitation and owning task.  Categorize results as:

- **blocker** — must be resolved or explicitly release-gated before an image
  is installed/distributed;
- **high** — privileged, safety or supply-chain exposure needing a scoped
  decision before the affected feature/release;
- **tracked** — already bounded by a task/ADR but not yet completed; or
- **informational** — no remediation proposed from this audit.

Do not change application behavior, source URLs, credentials, safety settings
or networking merely because a string search finds something.  Every
remediation needs a separately authorized task and proportionate tests.

## Verification

The audit must use read-only source inspection and record the exact repository
heads examined.  It must not execute installer/update scripts, target binaries
or printer actions.  Static scans are leads, not proof of runtime behavior.

## Hardware verification

Not run.

## Initial known leads

- The legacy README still advertises a shared root password and historical
  xblax release/support links.
- The project-specific Klipper package is pinned to an `xblax/klipper` commit
  while its generated MCU firmware is safety-critical; source hierarchy and
  provenance need an explicit decision.
- AD5M-016, AD5M-018/023, AD5M-019, AD5M-025/027 and AD5M-026 already own
  parts of authorization, egress, update, removable-media and recovery risk.

## Findings — first source pass (2026-10-28)

Inspected heads: `ad5m-project` `4504d50fe3fe4d42f09215f631cfadd33b6a56cb`,
`ad5m_enhanced_mod` `85ff97d59a484115a37c63b5831f79a3698773d8`, and
canonical Buildroot `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de`.  These are
static observations, not evidence of runtime behavior or malicious intent.

| Rank | Evidence | Current control / limitation | Owner |
| --- | --- | --- | --- |
| blocker | `build_scripts/buildroot/configs/base` sets `BR2_TARGET_GENERIC_ROOT_PASSWD="klipper"`; the legacy README publishes `root` / `klipper`. | The shared credential is present in the image configuration, not merely stale prose. It is incompatible with a distributable LAN-accessible release. | AD5M-029 with AD5M-016. |
| high | `package/klipper/klipper.mk` fetches pinned commit `fc2aaae…` from `https://github.com/xblax/klipper.git` and builds mainboard/eboard MCU artifacts; local patches include timing behavior (`0003-Relax-MCU-TRSYNC_TIMEOUT-for-ffad5m.patch`). | Exact Git pin improves repeatability but does not establish fork provenance, patch delta, license/source equivalence, or safety behavior. Do not replace it with an upstream commit without a compatibility/safety audit. | AD5M-030. |
| high | `install_steps.sh` copies `/media/klipper_mod/*` into `/`, then calls automatic `S54mcu_update`; host installer invokes it with privileged mounts/chroot. | The existing installer is an explicit privileged USB code/content channel. AD5M-027 preflights archive/space only; it does not authenticate contents, isolate an overlay, stage a candidate, or separate MCU flashing. | AD5M-019, AD5M-025, AD5M-027. |
| high | `S54mcu_update` automatically flashes MCUs on version mismatch; `mcu_update.sh` delegates to stock `NationsCommand` and `IAPCommand`; `S55mcu_boot` resets/boots MCU serial paths. | Safety-critical and host-dependent; no target/hardware test was run.  The later source pass also found incomplete parallel-flash result accounting. | AD5M-019, AD5M-026 and AD5M-034; a focused patch audit is still required. |
| high | Moonraker config documents default bind `0.0.0.0:7125`; root services start HTTP on 4000/4001 and camera streaming on 8080. | Browser/API authorization and bootstrap remain proposed; listener presence is not proof of an exploit. | AD5M-004 and AD5M-016. |
| high | `S45ntpd` starts `ntpd -p pool.ntp.org`. | This is declared external egress outside the incomplete HelixScreen local-only profile. Time sync may be useful, but its release policy, failure behavior and privacy implications are not recorded. | New egress-policy decision or expansion of AD5M-018 required. |
| tracked | `S00klipper_mod`, mdev camera/storage handlers, installer/uninstaller and recovery marker paths execute as root or influence root boot. | AD5M-025 removed USB debug-script execution and inventories these paths; marker/recovery redesign remains incomplete. | AD5M-025. |
| tracked | Project packages fetch pinned Moonraker, Klippain-ShakeTune, backlight, Mainsail/Fluidd release assets, audio and six PyPI packages. Git pins and several archive `.hash` files exist; `mainsail`, `klippain-shaketune`, `klipper`, and backlight have no adjacent archive hash. | Git commit pins are immutable identifiers but not full source/provenance review; release assets need artifact-integrity records. Backlight has focused evidence in AD5M-020. | AD5M-008 / AD5M-020; further component inventory remains. |
| tracked | Runtime Python copies have component-level provenance notes. | Coverage is limited to files audited under AD5M-021; it is not a complete executable-source inventory. | AD5M-021. |
| informational | Tracked `.xz` framebuffer assets are used by installer/init UI screens; a local conversion script exists for device images. | They are opaque release inputs but not executable code by themselves. Need asset digests/provenance in the future release manifest, rather than a blanket ban on generated assets. | AD5M-019 / release manifest work. |

### Privileged entry-point map

- **Stock host before chroot:** `device_files/install/flashforge_init.sh`,
  `device_files/uninstall/flashforge_init.sh`, and host `etc/init/S00klipper_mod`.
  The latter mounts `/data`, constructs the chroot and handles USB/update and
  recovery markers.
- **Chroot init:** `init_chroot.sh` enables swap and runs `rcS`; root services
  start NTP, MCU update/boot, Klipper, Moonraker, Mainsail, Fluidd and camera.
- **Hot-plug:** mdev starts camera discovery for `video[0-9]`, invokes
  `modprobe` for kernel modaliases, and automounts `sd[a-z].*` read-only.
- **User/recovery actions:** `jailbreak`, `reboot-stock-system`,
  `remove-klipper-mod`, MCU update helpers and the privileged installer paths.

This map is an inventory, not an assertion that every entry point has a
defect.  The next audit pass must add exact code origins/licenses and patch
deltas, then turn only confirmed release gates into separately approved tasks.

## Completion pass (2026-10-28)

The completed source-only pass inspected project heads
`ad5m-project` `1c18e71c9ba366a4e3e436fafdbced8bab4eecd3`,
`ad5m_enhanced_mod` `85ff97d59a484115a37c63b5831f79a3698773d8`, and
canonical Buildroot `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de`.  Searches
were limited to tracked source files; no installer, target binary, build
output or printer was executed.

| Rank | Evidence | Current control / limitation | Owner |
| --- | --- | --- | --- |
| high | `build_overlays/common/etc/init.d/S45ntpd` unconditionally starts BusyBox `ntpd` against `pool.ntp.org`. | It is an independent runtime egress path, not HelixScreen telemetry.  The project has not yet chosen whether time sync is off by default, locally configurable, or supplied by a local network source. | AD5M-032. |
| high | `build_overlays/common/lib/modules/cbc.ko` and `md4.ko` are tracked ARM relocatable kernel modules.  The inspected tree contains no adjacent module source, build recipe, license file or digest/provenance record. | Their current SHA-256 identities are `63e6f2f…0e337d` and `aad8913…bc156`, respectively.  Presence is not proof of malicious behavior, but opaque kernel code cannot be accepted into a release without provenance or a deliberate removal decision. | AD5M-033. |
| tracked | Mainsail has neither a package license declaration nor an adjacent archive hash; Git packages (Klipper, Moonraker, Klippain-ShakeTune and backlight) pin commits but do not retain a source-archive digest. | Existing AD5M-008 closure verifies selected Python and Fluidd/archive inputs.  The remaining package/release manifest inventory is a reproducibility and supply-chain gap, not a reason to replace sources wholesale. | AD5M-008 / AD5M-019 / AD5M-030. |
| tracked | `printer_configs/moonraker.conf` has an explicit home-LAN first-claim profile: `0.0.0.0/0` trusted clients and wildcard CORS while no normal user exists, plus API keys and `force_logins: true`; `S70mainsail`, `S70fluidd` and camera service expose HTTP listeners. | The broad fresh-device setup state is an accepted owner-first UX decision, not a silent inherited default.  Once a normal user exists, pinned Moonraker's `force_logins` removes trusted-client fallback.  Runtime transition and persistence evidence remain under AD5M-016/029; CORS is not an authentication boundary. | AD5M-016 / AD5M-029. |
| informational | Package-local patches are small and named: Klipper C-helper build check/version fallback/TRSYNC timeout, Moonraker runtime-pip disablement, and ShakeTune path relocation. | Patch size or commit metadata is not a safety conclusion.  Their exact behavior and the fork delta remain AD5M-030's evidence-backed audit. | AD5M-030. |

The audit's deliverable is now complete: all confirmed source-level findings
have an owner, and no behavior was modified merely from static inspection.
AD5M-031 orders the resulting non-hardware work and physical sessions.  This
does **not** lift the release/install gate: the shared credential, unreviewed
MCU-producing Klipper source, destructive update path and unproven recovery
still prevent an installable/distributable claim.
