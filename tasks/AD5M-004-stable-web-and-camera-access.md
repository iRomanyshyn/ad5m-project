# AD5M-004 Stable web and camera access

Status: proposed

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

No implementation has started. Existing endpoint behavior was identified from
the repository; no service configuration was changed.

## Verification

Not run.

## Hardware verification

Not run.

## Remaining work

Inspect the current startup scripts and Moonraker webcam configuration before
selecting an approach.
