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

### Draft access model for review

This is a candidate design, not an approved configuration or implementation.

1. Give each printer a persistent, collision-resistant mDNS name.  Keep
   Moonraker's direct LAN endpoint as the canonical slicer endpoint:
   `http://<device-name>.local:7125`.  Do not make a proxy the only way to
   upload a file.
2. Enable Moonraker API keys and use the native Moonraker API key in slicers
   that support it.  This is the normal `X-Api-Key` HTTP mechanism; it is
   independent of CORS.  Keep `[octoprint_compat]` for legacy uploader paths:
   Moonraker's [integration documentation](https://moonraker.readthedocs.io/en/latest/external_api/integrations/)
   identifies that component for PrusaSlicer derivatives and Cura's OctoPrint
   uploader, while noting that most current slicers speak the native upload
   API.
3. Use Moonraker user login/JWT for the browser UI, with `force_logins` only
   after a user exists.  Avoid broad `trusted_clients` as an authentication
   substitute.  The unsolved prerequisite is a safe first-user/bootstrap flow
   (for example a one-time pairing secret available through a local console or
   physical UI); a release must not ship a shared default credential or leave
   the whole LAN trusted indefinitely.
4. Allow CORS only for exact supported UI origins such as
   `http://<device-name>.local`; slicer requests do not need a browser CORS
   exception.  Direct Moonraker operation should not trust forwarded headers;
   review the exact behaviour of the project's pinned Moonraker version before
   setting `use_xheaders`.
5. Treat a narrow, user-selected private-LAN trusted-client range as an
   explicit compatibility profile, not the secure default.  It can help older
   clients that cannot send an API key, but every host in that range receives
   full printer-control access.

### Reverse-proxy decision

Do **not** add a proxy in the first access-control change.  It adds another
privileged HTTP parser and does not authenticate Moonraker requests by itself.
Use it only if the product later needs same-origin `/webcam/` camera URLs or a
single public port.  In that case use one small, reviewed HTTP reverse proxy
(rather than custom CGI glue or a second application), keep authentication in
Moonraker, and explicitly test WebSocket upgrade, streaming uploads, maximum
G-code size, `X-Forwarded-*` handling, camera absence and proxy restart.

If a proxy becomes the sole public API path, Moonraker may bind to loopback;
otherwise retain direct TCP 7125 for slicers and do not treat proxy headers as
client identity.  The exact option names and security semantics must be
verified against the project's pinned Moonraker commit (`71f9e677…`), not just
current upstream documentation.

## Acceptance criteria

- No release variant seeds `trusted_clients: 0.0.0.0/0` or `cors_domains: *`.
- The approved browser entry point and supported WebSocket/API flow work from
  their documented local origins.
- Native Moonraker slicer upload with an API key and the retained
  OctoPrint-compatibility upload path are tested independently.
- Upgrade does not silently grant new remote control to a previously
  untrusted network client.
- Security, software-only, and physical-LAN test evidence are distinct.

## Verification

Not run.

## Hardware verification

Not run.
