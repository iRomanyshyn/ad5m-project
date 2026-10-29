# AD5M-017 Persistent device name and mDNS identity

Status: proposed

## Goal

Give each printer a persistent, collision-resistant local device name that
survives updates and can be used safely in documented browser, camera and
slicer addresses.

## Context

The image currently has the build-time hostname `AD5M`.  The chroot startup
script applies `/etc/hostname`, the installer deliberately persists that file,
and Moonraker's enabled `[zeroconf]` component advertises the hostname on the
local network.  There is no current first-boot generator or per-device naming
flow.  Two printers using the stock hostname can consequently contend for the
same `AD5M.local` name.

AD5M-004 needs a stable address so the web UI and camera do not depend on a
DHCP address.  AD5M-016 additionally needs the final name if a browser origin
is to be narrowly allow-listed instead of using a broad CORS wildcard.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Define a valid, readable, collision-resistant default name for a new
  installation, without Internet access or a cloud service.
- Persist the name in the existing static `/etc/hostname` location before
  Moonraker advertises it.
- Never silently replace a user-selected persistent hostname on update or
  migration.
- Define how the name is displayed and changed locally, and how the associated
  Moonraker CORS configuration follows a deliberate name change.
- Keep resource use and writes appropriate for the constrained printer; name
  generation must not become a boot-time service or recurring write.
- Record software-only collision, character-validation and upgrade tests, and
  distinct physical-LAN mDNS resolution tests.

## Non-goals

- Router, DHCP, DNS or Internet configuration.
- A claim that `.local` naming works on every client operating system without
  validation.
- Changing a previously selected hostname merely to make URLs look uniform.

## Proposed implementation

First select one of two product policies:

1. require the owner to select a hostname during a local setup flow; or
2. generate an installation-time name from a cryptographically random,
   hostname-safe suffix, persist it once, and let the owner rename it later.

The second option is the better starting candidate for a zero-config local
URL, but needs review of the target's available entropy and tools.  Do not
derive a public name solely from a hardware serial or MAC address without an
explicit privacy and collision review.  Do not append an arbitrary suffix only
after an mDNS collision: that would make a previously documented camera or
slicer address unstable.

Before implementation, map all consumers of the hostname: the init script,
Moonraker zeroconf, browser/CORS origin generation, static UI configuration,
camera URL defaults, and slicer documentation.  Treat the legacy `AD5M`
hostname as a migration case.  An upgrade must preserve an already customized
hostname and must not cause a service to keep advertising the old name after a
rename.

## Implementation notes

### 2026-10-27 persistence and mDNS audit

The current installer already has the right persistence primitive, but not the
right initial value.  On its first run it moves `/etc/hostname` to
`/mnt/data/.klipper_mod/static/etc/hostname` and replaces the original path
with a symlink.  The chroot init script applies that file with `hostname -F`
before starting the normal services.  Consequently, a hostname generated
before this first `setup_static_data /etc/hostname` call will persist across
ordinary re-installs; changing a build default alone will not repair a device
that has already persisted `AD5M`.

No separate mDNS daemon is needed for the candidate design.  The pinned
Moonraker source's `[zeroconf]` component defaults `mdns_hostname` to the OS
hostname, publishes a `_moonraker._tcp.local.` service with
`<hostname>.local.` as its server name, and refreshes its advertised addresses
when a listener bound to all interfaces receives a network-state change.  The
current overlay does not override `mdns_hostname`.  This is source evidence
only: a real LAN must still verify that the final name resolves on the clients
we support, both before and after a DHCP address change.

The concrete first-release candidate is an installation-time name of the form
`ad5m-<16 lowercase hexadecimal characters>`, generated from 64 bits of
cryptographically secure local randomness and written once to the existing
static hostname file.  It is a valid short DNS label, readable enough to show
and copy, avoids exposing a hardware serial or MAC address, and makes random
collisions negligible at project scale.  Entropy availability and the exact
small target-side generator must be checked in the built image before this is
implemented; it must fail safely rather than silently falling back to a
predictable suffix.

Existing installations, including ones whose persisted value remains `AD5M`,
must not be silently renamed.  A local owner-initiated rename needs one
validated operation that atomically writes the static hostname, applies the
runtime hostname, regenerates the narrow Moonraker browser-origin policy from
AD5M-016, and restarts/re-registers the affected services.  Validate a single
ASCII DNS label (lowercase letters, digits and interior hyphens; 1--63
characters) and reject names such as `localhost`, dots, whitespace and shell
metacharacters.  If any generated configuration fails validation, retain the
previous hostname and origin policy rather than leaving one updated without
the other.

## Acceptance criteria

- Two fresh installations select different valid names without network access.
- The selected name persists across a normal update and is not regenerated at
  every boot.
- An existing non-default hostname survives unchanged; the treatment of an
  existing `AD5M` hostname is explicit and requires owner action where needed.
- The documented UI, camera and slicer URLs use the name consistently, or
  state their exception explicitly.
- CORS configuration is updated safely or fails closed when a hostname changes.
- mDNS name-resolution evidence from a real LAN is recorded separately from
  software-only tests.

## Verification

### 2026-10-29 first-install generator

The base image now supplies `usr/libexec/ad5m-device-name`. On a fresh
installation only — when no persisted static hostname exists and the seed is
the generic `AD5M` value — `install_steps.sh` generates
`ad5m-<16 lowercase hexadecimal characters>` from eight bytes of
`/dev/urandom`, validates it, writes it before `setup_static_data` persists
`/etc/hostname`, and logs the chosen name. BusyBox `dd`, `od` and `tr` are
enabled in the tracked configuration; this was checked before choosing the
small helper.

If randomness or validation fails, the installer retains the visible generic
`AD5M` hostname rather than inventing a predictable identifier or attempting a
late destructive rollback. This is not a collision-safe success state; the
owner must rename it before treating mDNS as a stable identity. An already
persisted hostname, including a legacy `AD5M` hostname, is never silently
renamed by this logic.

`make -C ad5m_enhanced_mod validate` runs the generator against deterministic
eight-byte fixture input, checks valid/invalid labels, and statically asserts
that installation invokes it only before fresh static-hostname creation. The
test is host-side; no mDNS listener, DHCP lease, browser, camera or printer
was run.

### 2026-10-29 first-install discovery record

After successful setup, the installer writes the non-secret
`AD5M-ADDRESS.txt` file to the same mounted USB medium when one is available.
It reads the persisted hostname from static state, validates it with the same
helper, and records copy-ready current UI (`:4000`), Moonraker/slicer
(`:7125`) and camera (`:8080`) URLs. It does not contain a password, API key,
Wi-Fi credential, serial number or log dump. The installer log also records
the generated hostname.

This is the immediate discovery path, not the final UX: a future on-screen
setup/status flow (including HelixScreen when it is integrated) must display
and allow owner-initiated renaming of the current address. The USB record is
available only when the stock installer mounted the installation medium; DHCP
lease discovery remains the fallback. The static test confirms the record is
written only to a mounted medium and only for a validated hostname.

## Hardware verification

Not run.
