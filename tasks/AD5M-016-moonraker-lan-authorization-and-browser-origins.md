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
   exception.
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

### Pinned Moonraker `v0.9.3` audit (2026-10-27)

The exact source at `71f9e677b81afcc6b99dd5002f595025c38edc7b` was inspected
in a temporary clone; `git fsck --no-dangling` passed.  No binary was built or
run.  The source confirms that this version supports JWT user logins, a single
system API key, one-shot tokens, `force_logins`, native HTTP upload, and the
partial OctoPrint upload API.  The system API key is a device-wide credential,
not a separately revocable per-slicer token; regenerating it disrupts every
slicer using it.

There is a material constraint absent from current documentation assumptions:
v0.9.3 starts Tornado with `xheaders=True` unconditionally.  It has no
`use_xheaders` configuration option.  A direct network client can therefore
supply forwarding headers, so an IP-based `trusted_clients` list must not be
treated as protection for the public TCP 7125 listener.

The version's authorization order is useful for a secure direct-slicer mode:
it checks a JWT and then the API key before IP trust; once `force_logins: true`
and at least one normal user exists, it rejects the trusted-client fallback.
Thus the candidate steady state is:

```ini
[authorization]
enable_api_key: true
force_logins: true
# no LAN trusted_clients entries
cors_domains:
  http://<device-name>.local
```

This permits a browser to use a user login/JWT and a slicer to use the API key
over direct TCP 7125.  It is not a complete deployment design yet: the first
normal user cannot be created from an unauthenticated network client, while
`force_logins` is not effective until that user exists.  A safe bootstrap path
is now an explicit release gate.  Viable candidates are installer-time
provisioning, a local-console/SSH command after an explicit credential change,
or a later physical-screen pairing flow.  Do not open a temporary trusted LAN
range or ship a shared bootstrap password.

A proxy is consequently optional for steady-state authentication, but can
provide a safer compatibility boundary if it binds Moonraker to loopback and
sanitizes/replaces all client-provided forwarding headers before proxying.  It
still needs Moonraker JWT/API-key authorization and does not remove the
bootstrap requirement.

### Persistence and migration audit (2026-10-27)

The existing installer makes `/root/printer_data/database` persistent by
moving it below `/mnt/data/.klipper_mod/static` and replacing the live path
with a symlink.  The current update documentation independently lists the
Moonraker database among the few preserved items.  Moonraker `v0.9.3` stores
normal users and its one system API key in that database.  Therefore a normal
update should preserve an existing user's login and slicers' API key; it must
not solve an authorization migration by deleting or recreating the database.
This behaviour still needs an update-path test on a disposable image/chroot.

In contrast, the post-build script clears the image's staged
`/root/printer_data` and copies the seed `moonraker.conf` anew.  The security
policy in that seed is consequently release-controlled, while credentials are
per-printer persistent state.  Migration design must test all of these states:

1. a new installation with no normal user;
2. an existing installation with a normal user and API key; and
3. an existing installation that has the old permissive configuration but no
   normal user.

Case 3 is the dangerous edge case: applying `force_logins: true` with no
normal user does not activate the protection, yet removing all trusted clients
at the same time leaves no remote path to create the first user.  The approved
bootstrap must deliberately handle it without a shared credential, a
temporary LAN-wide allow-list, or destruction of the persistent database.

The exact CORS origin also cannot be left as a build-time placeholder when the
printer's persistent hostname is unique.  The implementation must either
generate and safely validate a per-device origin from that hostname, or use a
reviewed same-origin frontend.  Do not replace it with `http://*.local` merely
for convenience: v0.9.3 accepts it, but it is broader than the intended single
printer origin.

## Acceptance criteria

- No release variant seeds `trusted_clients: 0.0.0.0/0` or `cors_domains: *`.
- The approved browser entry point and supported WebSocket/API flow work from
  their documented local origins.
- Native Moonraker slicer upload with an API key and the retained
  OctoPrint-compatibility upload path are tested independently.
- The selected first-user bootstrap method is tested so no unauthenticated LAN
  client can claim a trusted identity with a forwarding header.
- Update-path tests preserve an existing normal user and API key, do not
  recreate the Moonraker database, and cover the legacy no-user state.
- The browser CORS origin is derived from the final per-device name or avoided
  with a reviewed same-origin design; no broad `.local` wildcard is added.
- Upgrade does not silently grant new remote control to a previously
  untrusted network client.
- Security, software-only, and physical-LAN test evidence are distinct.

## Verification

Not run.

## Hardware verification

Not run.
