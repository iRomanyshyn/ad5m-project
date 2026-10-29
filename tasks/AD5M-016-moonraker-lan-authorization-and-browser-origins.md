# AD5M-016 Moonraker LAN authorization and browser origins

Status: in progress

## Goal

Define and validate a least-privilege Moonraker authorization and CORS policy
that supports the project's local browser UI without granting every network
client privileged printer control.

## Context

The historical `printer_configs/moonraker.conf` disabled API keys, trusted
`0.0.0.0/0`, and permitted `cors_domains: *`.  The current personal-alpha
seed makes the first-claim policy explicit: API keys and `force_logins` are
enabled, while broad trusted-client access remains effective only until a
normal user exists and broad CORS remains a UI-compatibility setting.
Moonraker documents trusted clients as full API access and
reserves a global CORS wildcard for development, not production; here that
tradeoff is an intentional home-device UX choice, not an authentication claim.
The web/camera work in AD5M-004 must not silently broaden it.  See Moonraker's
[authorization configuration](https://moonraker.readthedocs.io/en/stable/configuration/)
and [authorization API](https://moonraker.readthedocs.io/en/latest/external_api/authorization/).

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Specify the supported local-network threat model and which clients may
  control the printer without a login.
- Make universal trusted-client and CORS wildcard behaviour an explicit,
  documented first-claim/home-LAN compatibility decision rather than an
  accidental inherited default.  After a normal user exists, authentication
  must not depend on client IP or CORS.
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
3. Use Moonraker user login/JWT for the browser UI, with `force_logins` after
   a user exists.  The personal/home-LAN first-claim model intentionally lets
   the first local client create that first user; it is not a cloud or
   enterprise enrollment product.  The database must make this a one-time
   transition, so a normal user is required thereafter and an update cannot
   silently reopen setup.
4. Treat CORS as browser-UI compatibility rather than authentication.  The
   personal-alpha first-claim profile may retain a broad compatibility setting
   while the final UI origin/hostname work is incomplete; slicer requests do
   not need CORS, and any later narrowing must not break normal home-LAN use.
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
normal user cannot be created from an untrusted network client, while
`force_logins` is not effective until that user exists.  The selected product
policy now deliberately accepts a first-claim LAN setup state for a fresh home
printer; it is bounded by persistence of the first normal user, not by a
shared bootstrap password or a claim that an unclaimed device is safe on a
hostile LAN.

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

Case 3 is the first-claim edge case: `force_logins: true` does not activate
until a normal user exists.  The selected home-device profile intentionally
retains LAN setup access for that state, then relies on the persistent database
to prevent it from returning after first enrollment.  It must not destroy the
database or silently reset the device to unclaimed during an update.

The exact CORS origin also cannot be left as a build-time placeholder when the
printer's persistent hostname is unique.  The implementation must either
generate and safely validate a per-device origin from that hostname, or use a
reviewed same-origin frontend.  Do not replace it with `http://*.local` merely
for convenience: v0.9.3 accepts it, but it is broader than the intended single
printer origin.

## Acceptance criteria

- The shipped first-claim configuration explicitly enables API keys and
  `force_logins`; its broad LAN first-claim state is documented as accepted
  only while no normal Moonraker user exists.  Its broad CORS compatibility
  setting is not mistaken for a steady-state identity mechanism.
- The approved browser entry point and supported WebSocket/API flow work from
  their documented local origins.
- Native Moonraker slicer upload with an API key and the retained
  OctoPrint-compatibility upload path are tested independently.
- The selected first-user bootstrap is tested as an explicit first-claim
  transition: a fresh device accepts setup, while a device with a normal user
  rejects unauthenticated control even when forwarding headers are supplied.
- Update-path tests preserve an existing normal user and API key, do not
  recreate the Moonraker database, and cover the legacy no-user state.
- Browser CORS behaviour is documented as compatibility, not authentication;
  any later per-device-origin or same-origin design must preserve ordinary
  home-LAN access rather than introduce a mandatory deployment ceremony.
- Upgrade does not silently grant new remote control to a previously
  untrusted network client.
- Security, software-only, and physical-LAN test evidence are distinct.

## Personal-alpha first-claim bootstrap (2026-10-28)

Source inspection of the pinned Moonraker authorization component confirms
that `force_logins` only rejects trusted-client fallback after at least one
normal user exists.  The first normal user is created through the ordinary
`POST /access/user` flow, and the system API key is a shared device-wide
credential stored in the persistent Moonraker database.  Neither is a safe
unauthenticated remote bootstrap primitive.

For the personal-alpha profile, the selected direction is instead:

1. a fresh printer may let the first LAN client create the first Moonraker
   user through the normal UI/API; this ordinary home-device race is accepted;
2. seed `force_logins: true` and retain the temporary broad trusted-client
   setup capability only because pinned Moonraker ignores `force_logins` until
   a normal user exists;
3. as soon as that user is written to the persistent database, the same
   configuration automatically requires user JWT or API-key authorization and
   no longer grants control merely because a client is on the LAN; and
4. updates and rollback preserve that database and must never delete users or
   intentionally recreate a first-claim state.  Deliberate recovery/reset is
   a separate owner operation, not an unattended fallback.

This keeps first setup as easy as opening the printer's page, without a cloud
account, pairing app, SSH prerequisite or shared default credential.  It does
not claim protection from a hostile local network during the unclaimed state;
the first claimant is the owner by product design.  The per-printer SSH
password remains useful for Linux administration and recovery, but is not the
Moonraker enrollment authority.

The `xheaders=True` source fact remains important after enrollment: a
`trusted_clients` range cannot be treated as a meaningful LAN-security
boundary on a direct listener.  The steady state must depend on Moonraker
credentials/API keys, not an IP allow-list.  The exact CORS origin is a UI
compatibility question, not the ownership proof; resolve the final hostname
and Mainsail/Fluidd origin port without relying on CORS as authentication.

Before implementation, prove in a disposable rootfs/mock that:

1. a fresh profile permits first-user creation and normal initial UI access;
2. immediately after a normal user exists, an unauthenticated request and a
   forwarded-address spoof attempt fail, while JWT browser login and slicer
   API-key access work;
3. an existing normal user and API key persist across update/rollback without
   re-enabling first-claim access; and
4. an explicit owner recovery/reset procedure is separately documented and
   cannot run accidentally during boot or update.

## Verification

2026-10-28: the AD5M seed configuration was changed to explicitly set
`enable_api_key: true` and `force_logins: true`, retaining the documented
first-claim trusted-client profile for a fresh device with no normal user and
the separate broad-CORS compatibility setting.  The integration repository's
`make validate` passed; its static check rejects a future seed that drops
either API keys or `force_logins`, or silently changes the first-claim policy.
This is configuration/source
evidence only.  No Moonraker process, browser, HTTP client, slicer,
WebSocket, update transition or printer was run.

2026-10-29: a portable `moonraker-auth-contract` host target was added to the
integration repository. It requires an explicit `MOONRAKER_SOURCE_DIR`, checks
that the source's `authorization.py` SHA-256 and the package pin both match
`71f9e677b81afcc6b99dd5002f595025c38edc7b`, then imports that exact component
with only database, event-loop, HTTP and cryptography *boundary* fakes. Against
an ephemeral in-memory table it proved these component-level outcomes:

1. the configured trusted range is accepted while the table contains only
   Moonraker's API-key pseudo-user;
2. creating a normal user persists it and immediately makes an unauthenticated
   request fail with HTTP 401, including when it carries a forged
   `X-Forwarded-For` header;
3. the API-key and JWT control-flow branches authenticate after that
   transition; and
4. reconstructing the authorization component over the same table retains the
   user and API key, rather than reopening first claim.

The target has no target-architecture dependencies and does not open a TCP
listener, start Tornado/Moonraker, exercise a real Ed25519 implementation,
perform browser CORS/WebSocket negotiation, upload from a slicer, or invoke an
installer. It is therefore stronger than a textual source assertion but is not
evidence of end-to-end HTTP/Tornado behavior, production cryptography, a full
database migration, an image update, or physical-LAN behavior. Those remain
separate acceptance work.

## Hardware verification

Not run.
