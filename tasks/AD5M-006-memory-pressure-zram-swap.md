# AD5M-006 Memory pressure, zram and swap strategy

Status: in progress

## Goal

Measure real memory pressure on the AD5M platform and evaluate whether zram,
swap, service tuning, or a combination of them can improve reliability without
introducing unacceptable I/O wear, latency, or new failure modes.

## Context

The Flashforge Adventurer 5M / 5M Pro platform has approximately 128 MB of RAM
and runs the Klipper Mod stack in a constrained embedded environment.

The project already treats Klipper and MCU communication as higher priority
than optional services such as UI extras, camera processing, timelapse,
analytics, and background maintenance.

Memory-related failures must not be addressed by blindly enabling swap. The
project should first establish whether the actual problem is sustained memory
pressure, transient spikes, service leaks, cache pressure, or another resource
bottleneck.

## Repositories affected

Expected:

- ad5m-project
- ad5m_enhanced_mod

Potential:

- buildroot-flashforge-ad5m

The final ownership depends on whether any implementation belongs in the
Klipper Mod external tree, service configuration, or the generic Buildroot
base.

## Questions to answer

- What is the real RAM usage at idle, during normal printing, during camera use,
  during slicing-related metadata processing, and during known heavy workflows?
- Does the system currently hit OOM conditions, reclaim stalls, or severe
  memory pressure?
- Which processes are responsible for the largest steady-state and peak memory
  consumption?
- Are there services that should be tuned, delayed, disabled, or assigned lower
  priority before adding compressed or disk-backed swap?
- Is zram available and practical with the current kernel/userspace?
- What compression algorithm and zram size provide the best latency/RAM tradeoff
  for this CPU?
- Would a small disk-backed swap area provide useful emergency headroom?
- What storage target, if any, is acceptable for persistent swap?
- How should Klipper and MCU-related processes be protected from swap-induced
  latency or OOM selection?

## Candidate approaches

Evaluate in this order rather than assuming all mechanisms should be enabled.

### 1. Measure and reduce avoidable memory use

- collect baseline and peak memory metrics;
- identify leaks and optional heavy services;
- reduce camera/timelapse/UI/background resource use where appropriate;
- review process priorities and OOM behavior.

### 2. zram

Treat zram as the first swap-like mechanism to evaluate because it avoids
persistent flash writes.

Measure:

- compressed ratio;
- CPU overhead;
- allocation size;
- swap-in/swap-out activity;
- latency impact during printing;
- behavior during memory spikes.

Do not assume that a larger zram device is automatically better; compressed
pages still consume physical RAM.

### 3. Internal-flash-backed swap

Evaluate only after real measurements justify it.

Explicitly consider:

- write amplification;
- internal flash endurance;
- swap churn;
- latency under I/O pressure;
- interaction with logging and package/storage writes;
- behavior when storage is nearly full.

Persistent swap on internal flash must not be enabled by default without a
clear durability and reliability rationale.

### 4. USB-backed swap

Evaluate separately.

The project has historical reports of unreliable USB device detection, so USB
swap must not become a critical boot or runtime dependency.

If tested, failure cases must include:

- USB device absent at boot;
- device disconnect during operation;
- slow or failing media;
- filesystem/device corruption.

## Requirements

- Measure before changing defaults.
- Keep Klipper/MCU communication higher priority than optional services.
- Do not use swap as a substitute for fixing a memory leak or unnecessary
  background workload.
- Any zram/swap configuration must fail safely if unavailable.
- The printer must remain bootable and usable with swap/zram disabled.
- Do not create a hard dependency on removable USB storage.
- Keep persistent write amplification in mind when evaluating internal storage.
- Distinguish software-level stress tests from real-print hardware validation.
- Record exact configuration and measured results for each experiment.

## Useful metrics

Where supported, collect and compare:

- /proc/meminfo;
- free;
- process RSS/PSS;
- /proc/pressure/memory;
- vmstat;
- swap usage and swap-in/swap-out;
- zram compression statistics;
- OOM/kernel messages;
- CPU load during compression;
- storage I/O during swap activity;
- MCU/Klipper timing or timeout symptoms.

Prefer lightweight collection that does not itself materially distort the
resource-constrained system.

## Non-goals

- Enabling swap by default before measurement.
- Using an external Raspberry Pi merely to avoid understanding local memory
  behavior.
- Treating USB storage as mandatory hardware.
- Claiming that memory tuning fixes unrelated MCU, USB, storage, or network
  faults.

## Acceptance criteria

For the investigation phase:

- baseline memory behavior is documented for representative workloads;
- any observed OOM or pressure events are tied to evidence;
- the largest memory consumers and avoidable workloads are identified;
- zram feasibility is measured rather than assumed;
- internal-flash and USB-backed swap risks are explicitly evaluated;
- a recommended default is documented, including a justified recommendation of
  "no swap" if measurements show that is best.

For any future implementation:

- configuration is reproducible;
- it fails safely when the mechanism is unavailable;
- before/after measurements are recorded;
- printing reliability does not regress in physical testing;
- hardware testing is recorded separately from software validation.

## Implementation notes

### 2026-10-28 static baseline audit

Contrary to the original assumption that swap was only a candidate, the
current installer already creates a persistent **128 MiB file-backed swap**.
`install_steps.sh`, which runs inside the chroot during installation, calls
`fallocate -l 128M /mnt/swap`, `mkswap /mnt/swap`, and `chmod 0600`.  At this
stage `/mnt` is the chroot's own directory, so the resulting file is under
the installed chroot on the host `/data` partition; it is neither zram nor a
USB swap device.  On every chroot start `init_chroot.sh` sets
`vm.swappiness=15` and runs `swapon /mnt/swap`; the normal stop path calls
`swapoff /mnt/swap`.

The BusyBox configuration includes `mkswap`, `swapon`, and `swapoff`, but no
project-owned input selects `earlyoom`, `zramctl`, a zram package, a zram
kernel configuration, or an explicit OOM-score policy.  The AD5M kernel is
provided by the vendor host rather than a tracked project kernel config.
Consequently, static source review cannot establish whether its running kernel
has `CONFIG_ZRAM`, which compression algorithms exist, whether the current
swap file activates successfully, or whether it is used under real pressure.

This is an existing behavior, not approval of it.  The installer neither
checks free space nor checks the results of `fallocate`, `mkswap`, and
`swapon`; its shell flow continues if those commands fail.  Startup therefore
remains non-blocking when the file is unavailable, but an operator has no
recorded reason for the missing headroom.  The file also occupies internal
`/data` storage alongside the chroot, logs, G-code and update state, so its
write endurance and contention must be measured before changing its size,
enabling a second swap mechanism, or calling this configuration reliable.

The first hardware measurement must inventory the existing mechanism before
experimenting: record `cat /proc/swaps`, `free`, selected `/proc/meminfo`
fields, `vmstat`, `sysctl vm.swappiness`, process PSS/RSS via the already
shipped `smem`, and relevant kernel messages at idle and through one
representative print/camera/timelapse workflow.  If zram is considered,
record the running kernel's zram capability and compression choices first;
do not add or load a module solely to make the measurement pass.  Collection
must be read-only, bounded, and kept separate from heater, motion, MCU or
firmware operations.

### Decision direction

Do not alter the current file-backed swap solely from static inspection: on a
128 MiB device, removing emergency headroom before measuring could turn an
unknown degradation into an OOM failure.  It remains a **temporary baseline**,
not an endorsed default.  In particular, do not create a second default swap
layer on top of it.

The preferred experiment, if the vendor kernel demonstrably supports it, is a
bounded **zram-only** trial that disables the file-backed swap for that trial
and compares the same workloads.  zram avoids persistent flash writes, but
its compressed pages and CPU cost still consume scarce RAM/CPU; its size and
compression setting must be selected from observed pressure, not from a
generic 64/128 MiB rule.  A successful zram result is a candidate to replace,
not supplement, the internal-flash swap.

Decision after the measurements:

- If the present file swap has no meaningful use and no OOM/pressure evidence,
  remove it rather than reserving storage by habit.
- If memory pressure is real and zram is available with acceptable print-time
  latency, use one bounded zram configuration and retire the file swap.
- If zram is unavailable or measurably harms printer responsiveness, retain a
  small file-backed fallback only with explicit free-space checks, activation
  result logging, and evidence that its write/latency cost is acceptable.

USB-backed swap remains out of the default design.  Any chosen mechanism must
be optional at boot: Klipper Mod must start normally when it is absent or
cannot be activated, while leaving an actionable local log rather than
silently pretending that headroom exists.

### USB storage and internal-install research variant

An explicitly provisioned USB storage device is worth evaluating as an
**opt-in research variant**, including a modest physical modification that
routes USB internally and secures a fast, suitable device inside the printer.
It could reduce write pressure on the `/data` partition and avoids relying on
an externally dangling flash drive.  It is still removable-style storage from
the operating system's perspective, so an internal location does not make it
a boot requirement or erase its failure modes.

Before any implementation, evaluate the candidate device and modification for
power draw and startup sequencing, USB bandwidth/contention with the camera,
thermal exposure, vibration/strain relief, serviceability, electrical safety,
and recovery after a loose connector or a failed device.  The test plan must
include absent-at-boot, delayed discovery, live disconnect, I/O error,
full-device, filesystem/swap-header corruption, cold boot, warm reboot and
normal shutdown.  A printer must remain able to start Klipper Mod and print
with this device unavailable; no service may block indefinitely while waiting
for it.

Do not activate arbitrary USB media by kernel-assigned `/dev/sdX` name, label
alone, or a broad hot-plug rule.  Provisioning must be an explicit local
operator action and bind the chosen device to a stable identity and expected
capacity before activation.  It must never choose the camera, installation
media or an unknown user's drive.  The design also needs a clear disclosure:
swap can contain sensitive process memory, so an externally accessible drive
increases physical-data exposure unless a separately reviewed, usable
encryption/key-lifecycle design exists.

If measured, compare this option against both no-swap and zram-only using the
same workloads, recording swap I/O, print-time latency and the failure cases
above.  A favourable result may justify a documented user-enabled profile;
it does not justify making internal USB swap the production default.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

1. On a physical AD5M, verify whether the existing 128 MiB file activates and
   collect the bounded baseline described above across representative
   workloads.
2. Identify the actual `/data` filesystem capacity/free-space margin and
   write activity, including contention with logs, G-code and update staging.
3. Establish the vendor kernel's zram capability and only then compare a
   measured zram experiment against the present file-backed baseline.
4. Decide whether to retain, reduce, remove or replace the existing swap
   mechanism; do not enable two swap mechanisms by default without evidence.
5. If any policy changes, add explicit failure reporting and software checks,
   then run separately approved physical-print validation.
6. Only after those baselines, assess an explicitly provisioned internally
   mounted USB-storage profile against its mechanical, power, privacy and
   disconnect-recovery criteria; it remains user-enabled rather than default.
