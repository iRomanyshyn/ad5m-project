# AD5M-004 Stable web and camera access

Status: in progress

## Goal

Provide a documented primary web interface that does not require users to
enter Fluidd's non-standard port `4001`, and make camera access survive normal
printer IP-address changes without manually rewriting client configuration.

## Context

The current Klipper Mod publishes Mainsail on port 4000, Fluidd on port 4001,
Moonraker on 7125, and ustreamer on 8080. Camera documentation currently
requires manual client configuration. Direct IP-address URLs become stale when
DHCP assigns a new address.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Select and document one primary browser entry point that does not require
  `:4001` in ordinary use.
- Preserve an explicit, secure policy for Moonraker and web-interface access;
  do not accidentally expose privileged APIs merely to simplify the URL.
- Provide a stable camera URL or discovery mechanism that does not embed a
  mutable raw printer IP address in user configuration.
- Evaluate same-origin camera proxying, a stable local hostname such as mDNS,
  and other low-resource alternatives before choosing an implementation.
- Keep resource consumption suitable for the AD5M and avoid introducing an
  unnecessary daemon.
- Preserve user camera settings across upgrades where applicable.

## Non-goals

- Changing a user's router, DHCP configuration, or DNS without their consent.
- Publishing unauthenticated printer control to the Internet.
- Requiring users to run an external cloud service.
- Replacing Moonraker's authorization model.

## Proposed implementation

First map the current BusyBox HTTP, Mainsail, Fluidd, Moonraker, and ustreamer
endpoints. Compare a single-port/same-origin design against a local hostname
strategy, including authentication, CORS, websocket, camera-stream, update,
and recovery behavior. Implement only the smallest reviewed approach.

## Acceptance criteria

- The selected primary UI URL works without `:4001` in normal use.
- A previously configured camera remains reachable after a DHCP IP change,
  without editing its URL by hand.
- Access controls, CORS, websocket behavior, and camera exposure are tested
  and documented.
- Software testing and any physical-printer/network test are recorded
  separately.

## Implementation notes

### 2026-10-27 endpoint and discovery audit

The current image has three independent HTTP listeners:

| Component | Listener | Current role |
| --- | --- | --- |
| Mainsail | BusyBox `httpd`, TCP 4000 | static primary UI candidate |
| Fluidd | BusyBox `httpd`, TCP 4001 | static alternate UI |
| ustreamer | TCP 8080 | USB-camera stream and snapshot |
| Moonraker | TCP 7125 | API and WebSocket |

Neither BusyBox site config contains a reverse-proxy rule, so the commented
`/webcam/?action=stream` example in `moonraker.conf` is not currently exposed
by the Mainsail or Fluidd listener.  A relative Moonraker webcam URL becomes
useful only after a listener on the frontend host actually serves that path;
it is not itself a proxy.

Moonraker's configured `[zeroconf]` component and the already selected
`python-zeroconf` dependency can announce the printer on the local network.
This makes an mDNS name a lower-resource candidate than adding a proxy daemon:
serve the selected static UI on TCP 80 and document a stable
`http://<device-name>.local/` entry point; configure ustreamer as
`http://<device-name>.local:8080/...`.  The mDNS name must be persistent and
unique, not the shared build-time `AD5M` hostname.  A first-boot naming design
and a real LAN-resolution test are still required before selecting this
approach.  AD5M-017 now scopes that dependency; there is currently no
per-device name generator in the image.

Moonraker's [webcam API](https://moonraker.readthedocs.io/en/latest/external_api/webcams/)
supports complete URLs or paths relative to Moonraker's host, and stores
user-created webcam entries in its database.  A package must not overwrite
that database on upgrade.  A project-owned, config-sourced default is
acceptable only if it uses the final stable device name and has an explicit
migration/override policy.

### 2026-10-27 port-80 implementation audit

No additional HTTP daemon is necessary for the smallest browser-URL change.
The image already runs Mainsail using one BusyBox `httpd` process on TCP 4000;
the BusyBox configuration includes HTTPD with TCP 80 as its default feature.
The Mainsail init script can therefore become the single static site listener
on TCP 80.  No other project init script currently claims that port.  Fluidd
may remain an explicitly secondary listener on 4001 during migration, but it
must not be advertised as the primary UI.

This candidate intentionally does not proxy Moonraker or ustreamer.  Mainsail
would retain its direct Moonraker configuration on TCP 7125, while the camera
would retain its independent TCP 8080 endpoint.  The resulting documented
addresses would be `http://<device-name>.local/` for the UI and
`http://<device-name>.local:8080/...` for the camera.  This achieves the
no-`:4001` requirement with one fewer long-running process than a proxy design,
but it is still blocked on AD5M-016's reviewed authentication/CORS migration
and AD5M-017's persistent name.

Before changing the init script, validate an image/chroot with the selected
UI: static asset delivery on port 80, Moonraker HTTP and WebSocket calls from
the exact browser origin, UI login/bootstrap, direct native slicer upload,
retained OctoPrint-compatibility upload, camera absence, and clean service
restart.  This is a software-only candidate; physical LAN and camera tests
remain separate.

The existing Moonraker authorization defaults are unsafe as a baseline for
this work: they disable API keys, trust `0.0.0.0/0`, and allow CORS from `*`.
They must not be carried forward or widened merely to make the browser URL
convenient.  AD5M-016 records the separate authorization and browser-origin
decision that blocks implementation of this task.

## Verification

Repository and official-documentation inspection only.  No listener,
Moonraker setting, camera setting, hostname, router, or printer was changed.

## Hardware verification

Not run.

## Remaining work

1. Define and review AD5M-016's Moonraker authorization/CORS migration before
   changing browser or API endpoints.
2. Compare the TCP-80 + persistent mDNS design against a same-origin proxy,
   including measured storage/RAM cost, WebSocket behavior, and recovery when
   the camera is absent.
3. Design a persistent, collision-resistant device name and a camera-default
   migration that never overwrites a user's webcam database entry.
4. Build static/service checks, then test browser access, mDNS resolution,
   camera recovery and access control on a physical LAN separately.
