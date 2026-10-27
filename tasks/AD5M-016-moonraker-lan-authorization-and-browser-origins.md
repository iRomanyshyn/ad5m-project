# AD5M-016 Moonraker LAN authorization and browser origins

Status: proposed

## Goal

Define and validate a least-privilege Moonraker authorization and CORS policy
that supports the project's local browser UI without granting every network
client privileged printer control.

## Context

The current `printer_configs/moonraker.conf` disables API keys, trusts
`0.0.0.0/0`, and permits `cors_domains: *`.  Moonraker documents trusted
clients as full API access and reserves a global CORS wildcard for development,
not production.  The web/camera work in AD5M-004 must not preserve or broaden
these defaults simply to make a UI URL work.  See Moonraker's
[authorization configuration](https://moonraker.readthedocs.io/en/stable/configuration/)
and [authorization API](https://moonraker.readthedocs.io/en/latest/external_api/authorization/).

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Specify the supported local-network threat model and which clients may
  control the printer without a login.
- Replace the universal trusted-client and CORS wildcards with explicit,
  documented rules that support the selected same-device UI, loopback, and
  only any intentionally supported external browser origins.
- Decide whether local users authenticate, how credentials/API keys are
  initialized and rotated, and how an upgrade preserves existing access
  without silently opening control to the LAN.
- Keep Moonraker's API and WebSocket access working for supported local UI
  deployments without exposing them to the Internet.
- Test authorization, CORS, WebSocket and upgrade/migration behaviour in a
  non-printer environment where possible; record physical-LAN tests
  separately.

## Non-goals

- Configuring a user's router, firewall, DNS, VPN, port forwarding or cloud
  account.
- Claiming that a browser UI provides Internet-safe remote access.
- Changing Klipper heater, motion, MCU or printer safety configuration.

## Proposed implementation

First inventory every shipped browser origin and local integration.  Select an
explicit Moonraker policy and a backward-compatible migration/rollback plan.
Only then change the seed configuration and add static tests that reject global
wildcards from release variants.

## Acceptance criteria

- No release variant seeds `trusted_clients: 0.0.0.0/0` or `cors_domains: *`.
- The approved browser entry point and supported WebSocket/API flow work from
  their documented local origins.
- Upgrade does not silently grant new remote control to a previously
  untrusted network client.
- Security, software-only, and physical-LAN test evidence are distinct.

## Verification

Not run.

## Hardware verification

Not run.
