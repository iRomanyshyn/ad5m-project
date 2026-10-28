# AD5M-032 System network-egress policy

Status: proposed

## Goal

Define every non-HelixScreen outbound network capability in the AD5M image and
make its default, configuration owner, failure behavior and release test
explicit.

## Context

AD5M-028 found that `S45ntpd` unconditionally starts BusyBox NTP against
`pool.ntp.org`.  That traffic is distinct from HelixScreen telemetry, update,
crash or debug-bundle transport; an application-local policy cannot govern it.
The image also contains general network-capable tools, but their mere presence
does not establish an outbound service.

Time synchronization can be useful for logs, TLS and diagnostics.  It must
not be silently treated as mandatory Internet connectivity, leak a device
identity through a third-party default, or block printer operation when a
network/time source is unavailable.

## Requirements

- Inventory every boot-time or scheduled outbound endpoint, protocol and
  initiating service in the selected release profile; distinguish it from a
  tool merely capable of networking.
- For each enabled path, specify default state, local configuration owner,
  data sent, DNS dependency, retry/backoff, log behavior and offline failure
  behavior.  A printer must remain locally usable without external egress.
- Decide NTP policy explicitly: disabled by default, user-configured server,
  LAN-local default, or another evidence-backed option.  Do not hard-code a
  public pool merely for convenience.
- Keep local Moonraker/browser/camera/slicer traffic distinct from third-party
  egress.  Do not accidentally block DHCP, local DNS/mDNS or user-selected
  LAN time service while pursuing a local-only profile.
- Add source/static tests for selected service configuration and release
  inventory.  Target packet capture is later hardware/network evidence, not a
  substitute for the design.
- Align with AD5M-018/023 for HelixScreen and AD5M-019 for project update.
  This task must not re-enable upstream self-update, telemetry, crash upload
  or debug-bundle upload.

## Non-goals

- A general firewall feature, cloud connectivity, remote access or a claim
  that LAN-only traffic makes an insecure API safe.
- Changing user router/DNS/VPN configuration.

## Verification

Not run.  The current evidence is static inspection of the tracked
`S45ntpd` script only; no network traffic or printer was observed.

## Hardware verification

Not run.  The final chosen profile needs target-level packet capture on an
isolated test network before release.
