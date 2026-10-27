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

Not run.

## Hardware verification

Not run.
