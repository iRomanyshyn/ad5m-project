# AD5M-032 System network-egress policy

Status: in progress

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

## Static service inventory — 2026-10-28

The entries below distinguish a service that initiates a network connection
from a local listener or a tool merely capable of networking.  They are based
on the checked project configuration and startup scripts only; no packet was
captured and no target process was started.

| Path | Current source behavior | Egress assessment / owner |
| --- | --- | --- |
| `S45ntpd` | Starts BusyBox `ntpd` at every chroot boot with `-p pool.ntp.org`; DNS and public NTP traffic are implicit. | Confirmed boot-time third-party egress.  This task owns removal/replacement. |
| iwd and DHCP/network configuration | Network identity/configuration persists under `/var/lib/iwd`, `/etc/network/interfaces` and related paths. | User-selected LAN connectivity, not evidence of an Internet service.  Must remain usable for normal local networking. |
| Moonraker | `S65moonraker` starts the local API.  The checked `moonraker.conf` has no project `update_manager`, announcements or MQTT section. | No project-configured automatic outbound endpoint was found, but package/default behavior remains subject to AD5M-016/018 source review. |
| Fluidd, Mainsail, ustreamer | Boot starts local HTTP listeners on 4001, 4000 and 8080 respectively. | Inbound LAN exposure, not egress.  Their authorization/binding belongs to AD5M-016/023 and camera work. |
| Build/package fetch URLs | Buildroot recipes cite GitHub, PyPI and release URLs. | Build-host supply-chain traffic only; not target-image runtime behavior.  AD5M-008 owns source closure. |
| BusyBox networking tools and `crond` | The image configuration contains tools such as `wget`, `curl`, `telnet`, `ssh` and `crond`.  No project boot/cron job invoking them for egress was found in the inspected overlay. | Capability is not an enabled service.  A release inventory must nevertheless reject unexpected boot/cron network clients. |

## Proposed first-alpha policy

1. **Offline is the default.**  The image must boot and provide local printer
   functionality without DNS or a route to the Internet.  No boot service may
   contact a public endpoint by default.
2. **Disable public-pool NTP by default.**  Preserve fake-hwclock's last known
   time for offline operation.  A later explicit owner action may enable time
   synchronization only with a configured LAN/local server or a specifically
   chosen server, with the configured hostname/address visible to the owner.
   Failure must be quiet, bounded and never delay printer control.
3. **Separate local traffic from egress.**  DHCP, local DNS/mDNS if enabled,
   owner-selected Wi-Fi and browser/slicer/camera traffic remain distinct from
   a third-party connection.  The policy must not force a device off the LAN
   merely to disable Internet traffic.
4. **No hidden feature escape hatch.**  HelixScreen telemetry, crash upload,
   debug bundle upload and upstream self-update remain disabled under
   AD5M-018/023.  Moonraker update/announcements/MQTT or any new scheduled
   client needs its own reviewed configuration and a documented opt-in.
5. **Auditable configuration.**  The release manifest/inventory names every
   boot-time or scheduled initiating service, endpoint class and configuration
   file.  A static test rejects public endpoint literals and unmanaged cron or
   init clients in the selected profile.

This policy does not require a general-purpose firewall in the first alpha.
If later firewalling is considered, it must be an additional containment layer
and cannot replace removing unwanted application behavior.

## Implementation boundary

- Do not merely change `pool.ntp.org` to another hard-coded public hostname.
  Make the service disabled-by-default, then provide an explicit local
  configuration path with validation and safe defaults.
- Do not infer package runtime behavior from a grep result.  Audit enabled
  Moonraker/HelixScreen feature code and the final generated configuration
  before declaring the inventory complete.
- Test source configuration and service enablement on the development host;
  later use an isolated target network and packet capture to verify both
  offline behavior and each opted-in path.

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

2026-10-28 static audit completed for project startup scripts and checked
configuration: the public NTP client is confirmed; no other project-configured
boot-time third-party endpoint was found.  This is not packet-capture proof,
does not cover opaque stock-host behavior, and does not establish runtime
defaults inside every bundled application.

## Hardware verification

Not run.  The final chosen profile needs target-level packet capture on an
isolated test network before release.
