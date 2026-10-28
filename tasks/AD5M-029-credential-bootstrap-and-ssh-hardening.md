# AD5M-029 Credential bootstrap and SSH hardening

Status: in progress

## Goal

Eliminate the shared `root:klipper` image credential and define a recoverable,
per-printer first-access flow that does not silently open printer control to a
LAN.

## Context

AD5M-028 confirmed that the shared credential is both published in the legacy
README and configured in the image through
`BR2_TARGET_GENERIC_ROOT_PASSWD="klipper"`.  The device also runs LAN-facing
services and persists selected SSH/network state.  Removing the password
without a bootstrap/recovery design could lock out the owner; retaining it is
not acceptable for a releaseable image.

Moonraker's browser/API identity is a related but distinct boundary.  This
task must align with AD5M-016 rather than assuming an SSH password solves
browser authorization.

## Requirements

- Never ship a shared, predictable root password or packaged SSH private key.
- Define a per-printer bootstrap mechanism with clear local ownership proof,
  bounded lifetime and safe retry/recovery behavior.  It must work without
  cloud, a third-party relay or an external network service.
- Preserve or rotate existing authorized keys deliberately across a project
  update; do not overwrite network/SSH identity merely to reset access.
- Limit root remote access to the minimum required transport and audience;
  document password/key policy, host-key handling and failure recovery.
- Integrate the first-user flow with AD5M-016's Moonraker authentication
  bootstrap without a temporary LAN-wide trusted-client exception.
- Provide static configuration checks and target-level tests before release;
  do not test credentials on a production printer first.

## Non-goals

- Creating a cloud account, public remote-access relay or a universal recovery
  password.
- Changing motion, heater, MCU or update logic.

## Source constraints and alpha design (2026-10-28)

The current base configuration explicitly supplies `root:klipper` and enables
Dropbear.  The installer persists `/etc/dropbear`, `/root/.ssh` and the
Moonraker database below the per-printer static-data boundary.  This makes
host keys, authorized keys, normal Moonraker users and its system API key
durable across a chroot replacement, but it does not make the initial state
safe.

The pinned Moonraker source confirms that the first normal user is created by
its normal user API and that `force_logins` takes effect only once such a user
exists.  Therefore neither an empty database nor the system API key can be
treated as a first-owner proof.  The current generic Dropbear init script also
starts a network daemon before the existing Klipper/Moonraker services; it
needs an explicit profile rather than an inherited default password policy.

### Selected direction for the personal alpha

- Treat an ordinary home LAN as the normal use case, not as a hostile or
  enterprise-only environment.  The normal first-access path should be a
  plainly documented per-printer password and ordinary browser/SSH login;
  SSH keys, restrictive listener profiles and recovery controls must remain
  available without becoming mandatory rituals for a person who simply wants
  to use their printer.  Avoid security theatre: the essential baseline is no
  shared secret, no accidental unauthenticated control path, and a recoverable
  owner-controlled reset path.
- Ship no shared root password and no packaged private key.  Support both
  owner-selected per-device password access and SSH-key access; key-only is a
  supported profile, not a restriction imposed on an owner who reasonably
  prefers a password.
- The initial owner establishes access through a verified local console
  procedure (expected to be UART in the later physical session), a per-device
  operator-supplied public key from an ignored build input, or a per-device
  bootstrap password delivered through a local owner-controlled path.  Neither
  public key nor password belongs in Git or in a generic release asset.
- Before a key and normal Moonraker user exist, do not expose a temporary
  unauthenticated LAN setup route.  The bootstrap helper, if any, must be
  invocable only from the verified local path, must create its own bounded
  state, and must not print a reusable secret into ordinary logs.
- Retain the existing persistent SSH/Moonraker paths deliberately, with
  documented permissions and backup/redaction rules.  Rotation/revocation
  must require local owner authorization and must not silently delete working
  keys or users during update/rollback.

Password bootstrap must not rely on a vague "expired password" promise.  The
checked BusyBox configuration has shadow-password and `passwd` support, but
does not enable `chage` or `chpasswd`; and this audit has not established that
Dropbear turns a password-expiry condition into an SSH-side password-change
dialogue.  A blindly expired SSH password may instead be a lockout.  If a
first-login password-change experience is wanted, implement it as an explicit,
tested onboarding state: authenticate with the unique bootstrap password,
present only the bounded local setup action, require a successful `passwd`
change and durable completion marker, then retire the bootstrap secret.  It
is a convenience flow, not the security proof; secrecy/uniqueness and local
delivery of the initial password remain the proof.

### Open implementation proof

Before modifying the image, verify against the pinned Buildroot/Dropbear and
Moonraker sources that the selected configuration: locks password access while
allowing the intended key path, starts no public listener before bootstrap,
creates the first Moonraker user without direct database edits, and remains
recoverable after a failed bootstrap.  Exercise this on a disposable
non-printer rootfs/mock before the AD5M.  The actual console capability and
local ownership ceremony are P4 hardware evidence under AD5M-031/026.

### Buildroot/Dropbear capability audit — 2026-10-28

The canonical pinned Buildroot source supplies a concrete, reviewable route
for a key-only Dropbear profile: `BR2_PACKAGE_DROPBEAR_LOCALOPTIONS_FILE`
appends a project-controlled header to Dropbear's `localoptions.h`.  Its own
package logic already uses that mechanism for password-authentication settings
when PAM or `crypt()` availability requires it.  It is therefore preferable to
an untracked edit inside a downloaded Dropbear tree.

The current integration baseline is still explicitly unsafe:

- `build_scripts/buildroot/configs/base` sets
  `BR2_TARGET_GENERIC_ROOT_PASSWD="klipper"` and enables Dropbear.
- Buildroot's default root-login setting is password-capable when a root
  password is configured.  The base configuration does not explicitly record
  a stricter root-login policy.
- Buildroot's SysV `S50dropbear` sources `/etc/default/dropbear`, appends `-R`
  to generate missing host keys, then starts `/usr/sbin/dropbear`.  It does not
  itself turn a missing password into a safe ownership ceremony.
- The existing installer replaces `/etc/dropbear` with persistent state and
  also persists `/root/.ssh`, so host keys and `authorized_keys` have an
  intended per-printer storage boundary once initial ownership exists.

The proposed implementation sequence is therefore:

1. Add a small, committed AD5M Dropbear local-options/configuration profile
   with explicit password-authentication semantics, and reference it through
   the canonical Buildroot configuration rather than patching package output.
   The profile must support deliberate key-only and password-enabled modes.
2. Remove the shared root password only in the same reviewed change.  For the
   password mode, accept a per-device ignored build/install input or a
   verified local generation-and-delivery path; for key mode, accept a
   per-device ignored public-key input or local-console bootstrap.  No
   credential belongs in Git or in a generic release image.
3. Make the service's listen/address/port policy explicit in root-owned
   configuration.  Do not use a default listener plus a later firewall rule
   as the only access decision.
4. Test the built Dropbear binary and SysV path inside a disposable,
   non-printer rootfs: authorized key accepted in key-only mode; a unique
   bootstrap password accepted only in password mode; shared/expired/missing
   credentials rejected as designed; generated host key becomes persistent;
   malformed/missing bootstrap input fails closed; and existing persistent
   keys/password state survive a candidate/rollback transition.  Test an
   onboarding password-change flow explicitly; do not infer it from shadow
   file fields alone.
5. Only then change an AD5M image.  UART/local-console ownership evidence and
   real LAN reachability remain dedicated hardware tests, not substitutes for
   the disposable-rootfs checks.

This audit establishes configuration capability, not the selected Dropbear
runtime semantics for the final image.

### Per-printer password build input — 2026-10-28

The first small P2 implementation removes the committed
`BR2_TARGET_GENERIC_ROOT_PASSWD="klipper"` fragment.  The AD5M Buildroot
wrapper now requires a local, ignored file containing one raw SHA-256-crypt
(`$5$...`) hash before it initializes a variant output.  It rejects a missing,
multi-line, malformed, group-readable or world-readable input; it then writes
the Buildroot-required dollar-escaped value into the disposable generated
`.config`.  No clear-text password is passed to Buildroot or committed.

`build_scripts/create-root-password-hash.sh` is the ordinary home-LAN path:
it asks for the owner-selected password twice without echoing it, uses
`openssl passwd -5 -stdin`, writes only the hash below the ignored
`build_scripts/buildroot/configs/local/` directory with mode `0600`, and
refuses to overwrite an existing input.  An owner who stores the hash
elsewhere may select it with `AD5M_ROOT_PASSWORD_HASH_FILE`.

This fixes the shared SSH credential but does **not** yet prove a usable
Dropbear login or solve Moonraker/browser authorization.  In particular,
Moonraker's current broad trust configuration remains owned by AD5M-016 and
must not be mistaken for a secured web-control plane merely because the root
password is now unique.

## Verification

2026-10-28: source-only inspection established the current root-password,
Dropbear and persistent-state boundaries, the pinned Moonraker first-user
constraint, and the canonical Buildroot `localoptions.h` capability.

2026-10-28: `make validate` passed in the integration repository.  It checks
that a shared root-password fragment cannot return and that the local
SHA-256-crypt input/generator contract remains present.  `make
validate-config` passed against the recorded clean Buildroot revision in a
temporary output directory; it confirmed the known non-secret fixture hash is
correctly escaped in the final generated configuration.  The hash helper was
also exercised with a temporary password: it produced mode `0600`, a valid
SHA-256-crypt hash and rejected an overwrite.  No target rootfs, Dropbear
server, SSH client, Moonraker service or printer was run.

## Hardware verification

Not run.
