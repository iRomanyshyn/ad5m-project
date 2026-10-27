# AD5M-008 Package source closure and integrity

Status: complete

## Goal

Close the missing `python-msgspec` dependency reported by AD5M-007 and make
the affected release-source inputs fixed, auditable, and buildable.

## Context

AD5M-007 exposed an external Klipper recipe selection for a package absent
from the canonical Buildroot 2025.02 tree. The selection originated in
historical third-party work, which is untrusted evidence only under ADR-0001.
This work independently audited the exact source revisions; it did not fetch,
copy, or import code from that history.

The same review covered an unhashed Fluidd asset, a mutable GuppyScreen branch,
and Moonraker's actual dependency closure. The latter also attempted an
automatic runtime `pip` recovery, which would bypass the firmware supply-chain
review.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Implementation

### Klipper and `msgspec`

The annotated upstream tag `v0.13.0-285-gfc2aaae69-ffadm5` was resolved and
audited at immutable commit
`fc2aaae692b5f703420915390a3dcb8cd514c384`. `git fsck --no-dangling` passed
on the independently cloned source. The tag is not signed.

The exact source's `scripts/klippy-requirements.txt` labels
`msgspec==0.19.0` optional. `klippy/webhooks.py` catches `ImportError` and
uses the standard-library JSON encoder/decoder fallback. The external recipe
therefore no longer selects or requires an unavailable `python-msgspec`
package, and Klipper is now pinned to the reviewed commit rather than the
mutable tag name.

### Moonraker closure and runtime update boundary

Moonraker tag `v0.9.3` was resolved and audited at immutable commit
`71f9e677b81afcc6b99dd5002f595025c38edc7b`; `git fsck --no-dangling` passed.
The tag is not signed. Direct imports in core/server components require:

- `inotify-simple` for the core file manager;
- `libnacl` (and the system `libsodium`) for core authorization;
- `streaming-form-data` for the HTTP application;
- `importlib-metadata` at startup.

The required transitive packages are `smart-open` for
`streaming-form-data`, `wrapt` for `smart-open` (already provided by
Buildroot), and `zipp` for `importlib-metadata`. The six absent packages were
added to the external tree as minimal Buildroot recipes with fixed PyPI source
archives, SHA-256 records, SPDX license declarations, license-file hashes, and
their explicit target/host dependencies.

The source confirms that `apprise` (notifier), `ldap3` (LDAP), and
`preprocess-cancellation` (object processing) are loaded only for optional
configuration/functions; `pre-commit` is a development dependency. They are
intentionally excluded from the base image rather than being silently pulled
in. If an optional component lacks a module, the image's small Moonraker patch
logs the condition and returns `False`: it disables Moonraker's otherwise
automatic `pip` update/install path. This preserves the reviewed, offline
Buildroot dependency closure. Adding any optional component requires its own
fixed-source package review.

### Other source pins

- Fluidd v1.34.4 release asset is now hash-pinned as `fluidd.zip`:
  `582d5f32cbb5bdda9bbfe0dc39aeb251457e6595a9331e30d9c8c94dcbbf0588`.
  The binary asset has no license file; its independently inspected source tag
  v1.34.4 (`564242b151ea83d6f48ac5f027e506af7c177fe6`) declares GPL-3.0 in
  both `LICENSE` and `package.json`.
- GuppyScreen moved from `flashforge_ad5m` to reviewed commit
  `fdca5728001f7a6f6ce774dcfa3998f8b96e6878`; `git fsck --no-dangling`
  passed. The recipe now correctly records GPL-3.0+ and `LICENSE` rather than
  a nonexistent `COPYING` file. Its Git submodule Gitlink IDs are immutable
  fields in that commit, but their source provenance is not yet separately
  audited; this is tracked in AD5M-011. The rootfs recipe installs only
  `guppyscreen`, not its `update.sh`; the upstream update mechanism is tracked
  separately in AD5M-010.

### New PyPI packages

| Package | Version | SHA-256 | License |
| --- | --- | --- | --- |
| `inotify-simple` | 1.3.5 | `8440ffe49c4ae81a8df57c1ae1eb4b6bfa7acb830099bfb3e305b383005cc128` | BSD-3-Clause |
| `libnacl` | 2.1.0 | `f3418da7df29e6d9b11fd7d990289d16397dc1020e4e35192e11aee826922860` | Apache-2.0 |
| `smart-open` | 7.0.5 | `d3672003b1dbc85e2013e4983b88eb9a5ccfd389b0d4e5015f39a9ee5620ec18` | MIT |
| `streaming-form-data` | 1.15.0 | `891e011bcc18dceea06f64b20d65d79f77830bf6d746ae89d8debdce16afeea5` | MIT |
| `zipp` | 3.20.0 | `0145e43d89664cfe1a2e533adc75adafed82fe2da404b4bbb6b026c0157bdb31` | MIT |
| `importlib-metadata` | 8.2.0 | `72e8d4399996132204f9a16dcc751af254a48f8d1b20b9ff0f98d4a8f901e73d` | Apache-2.0 |

The sources were downloaded from PyPI, inspected locally, and each recipe's
archive and license-file hash was recorded. A current official-Buildroot
master checkout was also checked at `0c58b443d8328316906cbc177beae0d1c4c50ab5`;
it did not provide recipes for these packages, so an external-tree ownership
is appropriate.

### Portable generated configuration

The full build revealed two additional path defects. The wrapper now writes
absolute, checkout-derived paths into generated variant configs for both
Klipper MCU configs and the common/variant rootfs overlays. This follows the
same source-tree-vs-output-tree correction already made for the SDK path.

## Verification

- `git fsck --no-dangling` on the independently cloned exact Klipper,
  Moonraker, and GuppyScreen revisions.
- `git apply --check` for the Moonraker runtime-pip patch.
- `bash -n ad5m_enhanced_mod/build_scripts/build.sh` and `git diff --check`.
- Full canonical `lite` build and local package creation on 2026-10-27.
  Every new PyPI archive was SHA-256-verified by Buildroot; all six target
  packages installed. `streaming_form_data._parser` was cross-compiled as an
  ARM Python 3.12 extension.
- The resulting rootfs includes both `eboard` and `mainboard` Klipper
  `.bin`, `.elf`, and `.hex` firmware files; Klipper and Moonraker version
  files contain the reviewed commits; all six Python distributions are present.

Local generated artifacts (not release manifests and not committed):

| Artifact | SHA-256 |
| --- | --- |
| `rootfs.tar` (239 MiB) | `bd49c251201000459b3ee4ff8a78ce6f4c50658a4eb47bfda9fa3a536d2f18ef` |
| `chroot.tar.xz` (54 MiB) | `a7bcd7b103e00fbbec99b5a947bcb487233a815c901c10d73b6a5baef285661f` |
| `Adventurer5M-KlipperMod-9728a5d-dirty-lite.tgz` (55 MiB) | `a0a21115eb5c59b969cbe1cae6d8fc6b530ff0139bcff9d35d9a25c6b942efdd` |

Buildroot still prints no-hash warnings for Git-derived source archives. The
fixed, reviewed Git commit IDs above are the source identities; the generated
Git archive is a local download-cache representation, not an additional
upstream release artifact.

`utils/check-package` could not run on this workstation because its host
Python environment lacks `flake8`; this is a developer-tooling gap, not a
target-image dependency.

## Hardware verification

Not run. No artifact was installed on a printer; no MCU was flashed; and no
motion, heater, display, camera, network, or print behavior was tested.

## Remaining work

- AD5M-010 must decide and enforce the GuppyScreen update-button behavior
  before enabling that variant for a release.
- AD5M-011 must audit GuppyScreen's fetched submodule closure before enabling
  that variant for a release.
- AD5M-009 remains required before claiming that an FFmpeg-containing image
  is redistributable.
