# AD5M-006 Memory pressure, zram and swap strategy

Status: proposed

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

No implementation has started.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

After AD5M-001 establishes the canonical Buildroot integration, inspect the
current kernel configuration and runtime memory behavior, then define the
smallest useful measurement plan before experimenting with zram or swap.
