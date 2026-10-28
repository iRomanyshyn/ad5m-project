# 0005 Owner control and open-network policy

Status: accepted

## Context

The AD5M project is intended to be a maintainable, self-hosted modification
of an owner-operated printer.  Earlier planning used "local-only" as a
short-hand response to unreviewed outbound endpoints and automatic upstream
behaviour.  Taken literally, that would turn a safety boundary into a closed
appliance: it would unnecessarily prevent normal Internet time, voluntary
telemetry, optional support workflows, or an owner's deliberate integration.

The relevant risk is not that the printer can reach a network.  It is that
privileged software silently changes firmware, exports data, or claims an
owner's authority without an understandable configuration and recovery path.

## Decision

The product is **open, self-hosted, and owner-controlled**, not a
network-isolated or vendor-locked box.

- Normal local-LAN use by browser UI, camera and slicers is a supported
  product goal.
- Internet-facing capabilities such as time synchronization, telemetry,
  support export and update checking may be available when their endpoint,
  purpose, data scope, storage effects and disable/revoke path are clear to
  the owner.  They must not require a cloud account or a project-operated
  relay merely to use the printer.
- A safe default may avoid surprise third-party traffic during first boot, but
  must leave a documented, local owner-controlled way to enable a useful
  capability.  "Disabled by default" is not "unavailable by design".
- Owner-opt-in automatic **project** updates are a supported product goal once
  a verifiable recovery/update mechanism exists.  They may update a reviewed,
  compatible release bundle or explicitly independent components, but must
  have channels, immutable identities, a staged health check and rollback.
  Independent upstream self-updaters must not silently bypass that boundary.
- Automatic MCU flashing and automatic safety-critical printer-configuration
  changes remain out of scope.  This is an integrity and safety constraint,
  not an attempt to restrict the owner from installing a deliberately selected
  artifact.
- Authentication, consent, and artifact integrity must be proportionate to
  the capability.  They must not be used to force a proprietary service or
  prevent inspection, replacement or local administration by the owner.

An optional strict privacy/offline profile may be investigated later, but it
is not the baseline requirement that blocks normal HelixScreen or network
integration work.

## Consequences

AD5M-018, AD5M-023 and AD5M-032 retain their source findings, but their
"local-only" constraints are reinterpreted as requirements for transparent,
owner-controlled behaviour rather than a blanket ban on every outbound
feature.  Their implementation plans must expose deliberate configuration and
disable paths.  They must continue to reject hidden transport, unbounded data
export, shared credentials, and unattended privileged application of updates.

The existing opt-in NTP configuration is compatible with this decision: an
owner can configure a chosen server; the image does not silently choose a
public pool.
