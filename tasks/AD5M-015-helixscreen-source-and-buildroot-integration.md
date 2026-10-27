# AD5M-015 HelixScreen source and Buildroot integration

Status: in progress

## Goal

Create a first-class, reproducible HelixScreen package and service for the
AD5M external Buildroot tree, with a persistent configuration boundary and no
dependence on third-party AD5M installers or firmware mods.

## Context

HelixScreen is the project's sole planned touchscreen UI under ADR 0003.
The official upstream repository provides an AD5M-specific Buildroot target
(`PLATFORM_TARGET=ad5m-br`) and an install staging target, but its documented
AD5M installer is an imperative script for other firmware variants.  It must
not be run or imported into this project.

The candidate source is official stable tag
[`v1.0.6`](https://github.com/prestonbrown/helixscreen/tree/v1.0.6), resolving
to commit
[`11230009fa09b4509d0d05f482feaeef2e94c854`](https://github.com/prestonbrown/helixscreen/commit/11230009fa09b4509d0d05f482feaeef2e94c854)
from 2026-10-21.  It is GPL-3.0-or-later, carries a
`PLATFORM_TARGET=ad5m-br` path, installs assets under `/opt/helixscreen`, and
supports a separate writable `HELIX_CONFIG_DIR`.  The annotated tag has no
cryptographic signature; `git fsck --no-dangling` passed on the inspected
temporary Git object database.  This establishes source identity, not source
approval or a substitute for dependency closure.

At this candidate commit `ad5m-br` selects `fbdev`, `evdev`, dynamic target
libraries, and no SDL, GLES, or screensaver.  It does *not* set
`ENABLE_MOCKS=no`; the external package must do that explicitly and must pass
`HELIX_PACKAGING=1` (or an equivalent reviewed setting) to omit the
remote-control server.  No user-visible service is accepted until those build
properties can be inspected in its staged binary.

### Direct source inventory

The candidate has 15 direct Git submodules.  The table records Gitlink
identities and the upstream's own declared licenses; none is accepted yet.

| Path | Declared upstream URL | Gitlink commit | Pinned root-license evidence | Review state |
| --- | --- | --- | --- | --- |
| `lib/cpp-terminal` | `https://github.com/jupyter-xeus/cpp-terminal.git` | `48ae2f284084850901c45b6c10a9d68949c1b272` | MIT, `LICENSE` | root checked; code and vendored-content review pending |
| `lib/ftxui` | `https://github.com/ArthurSonzogni/FTXUI.git` | `b1527cb6966679da256409e577a0d44ae322b2e3` | MIT, `LICENSE` | root checked; code and vendored-content review pending |
| `lib/glm` | `https://github.com/g-truc/glm.git` | `a583c59e1616a628b18195869767ea4d6faca5f4` | Happy Bunny or MIT, `copying.txt` | root checked; code and vendored-content review pending |
| `lib/helix-xml` | `https://github.com/prestonbrown/helix-xml.git` | `9c8f87a231bc0e9914c2a3a61de0fc254d2f172d` | MIT, `LICENSE` | root checked; embedded expat and code review pending |
| `lib/libhv` | `https://github.com/ithewei/libhv.git` | `71770e04becaa149e0ef8ffc4d3900c5466ddddb` | BSD-3-Clause, `LICENSE` | root checked; vendored-content and code review pending |
| `lib/libjpeg-turbo` | `https://github.com/libjpeg-turbo/libjpeg-turbo.git` | `c85e6b905bf237038faa936dab160ebfc5da0344` | IJG and BSD-3-Clause, `LICENSE.md` | root checked; vendored-content and code review pending |
| `lib/libnl` | `https://github.com/thom311/libnl.git` | `bdf83151326e365f137fe0e36dc9b1b7aeb1cf33` | LGPL-2.1, `COPYING` | root checked; linkage and code review pending |
| `lib/linenoise` | `https://github.com/antirez/linenoise.git` | `452e3793858a15cc006bb3aa3ea06378eb98c5b5` | BSD-2-Clause, `LICENSE` | root checked; code review pending |
| `lib/lv_i18n` | `https://github.com/lvgl/lv_i18n` | `f6ac92ee2c31b57032fc6addc9aa15de2fee4568` | MIT, `LICENSE` | root checked; code review pending |
| `lib/lv_markdown` | `https://github.com/prestonbrown/lv_markdown.git` | `77ec2c0c91614a50575a44c39d8db52c057ae681` | MIT, `LICENCE.txt` | root checked; md4c and code review pending |
| `lib/lvgl` | `https://github.com/lvgl/lvgl.git` | `85aa60d18b3d5e5588d7b247abf90198f07c8a63` | MIT, `LICENCE.txt` | root checked; bundled-library and code review pending |
| `lib/openvdb` | `https://github.com/AcademySoftwareFoundation/openvdb.git` | `604ed3828033b84de74f2bc35b01bc655195cb8f` | Apache-2.0, `LICENSE` | root checked; code review pending |
| `lib/sdl2` | `https://github.com/libsdl-org/SDL.git` | `5d249570393f7a37e037abf22cd6012a4cc56a71` | zlib, `LICENSE.txt` | root checked; code review pending |
| `lib/spdlog` | `https://github.com/gabime/spdlog.git` | `14ab43ec0b9df6cf793c75785ec5f70b65f2f965` | MIT, `LICENSE` | root checked; fmt ownership and code review pending |
| `lib/wpa_supplicant` | `https://github.com/fluispotter/hostap.git` | `b5e326d24298ac52fba3c4e5ba5ae79780c78dc9` | BSD-3-Clause, `COPYING` and `README` | root checked; unused-feature decision and code review pending |

The upstream `COPYRIGHT` also declares bundled components, including minilzo
(GPL-2.0-or-later), quirc (ISC), and libraries nested in submodules.  This is
why checking only the root GPL notice is insufficient.

The table's license column is evidence from the named file at the pinned
commit, not a statement that every file beneath that source has been cleared.
In particular, root-license inspection exposed separate notices in LVGL,
libhv, Helix XML, SDL and libjpeg-turbo. Their shipped subset, notice handling,
linkage obligations and executable code remain part of the later review.

### Git-submodule closure audit (2026-10-27)

The root `.gitmodules` and every direct Gitlink were read from Git object
databases at the fixed commits in the inventory above. Each of the 15 pinned
objects exists and is a commit; `git fsck --no-dangling` passed for the root
and each direct source object database. At those exact commits, none of the
15 direct sources has a `.gitmodules` file or a mode-`160000` Gitlink.
Consequently the Git-submodule closure of this candidate is exactly the root
repository plus the 15 listed direct sources: there is no recursively fetched
Git source to add at this revision.

This is source identity and fetch-graph evidence, not dependency approval.
It does not cover vendored code, generated inputs, package-manager downloads,
or build-time network access. A future Buildroot recipe must retain the root
commit, use the canonical `GIT_SUBMODULES=YES` fetcher, and make a
no-network-after-fetch build an explicit verification requirement. Any
upstream update that changes the root commit or a Gitlink must repeat this
closure audit before it is eligible for packaging.

### Build-time network boundary audit (2026-10-27)

Static inspection separates upstream developer convenience from a package
build. The `install DESTDIR=...` target only stages already-built binaries and
tracked runtime data; it does not call a downloader. The candidate also tracks
the pre-generated MDI font C sources, MDI TTF, Latin font inputs, CJK runtime
font binaries, and MDI metadata cache needed by the inspected build graph.

The normal build is nevertheless not hermetic as written. Every generated MDI
font source depends on a root `.fonts.stamp`. That stamp is untracked; when it
is absent, its recipe checks the ambient host `PATH` and, if `lv_font_conv` is
present, runs `scripts/regen_mdi_fonts.sh`. That script currently uses a local
tracked MDI TTF and does not itself download, but letting a host-installed tool
rewrite generated sources makes the output host-dependent. Its absence merely
changes the recipe into a `touch`, which is not an acceptable security control.

Other developer targets are explicitly out of the package path: `install-deps`
can initialise submodules and run `npm install`; `update-mdi-cache` curls a
moving `master` branch; and `regen-text-fonts` downloads CJK inputs from a
moving `main` branch when they are absent. The CJK regeneration stamp is not
part of the ordinary target dependency discovered here, but that does not make
the target safe to call. No project recipe may invoke any of those targets,
the upstream Docker helpers, or an asset regeneration script.

Before packaging, add a small reviewed Buildroot-only adaptation which makes
the pre-generated tracked fonts authoritative and disables the `.fonts.stamp`
generator unconditionally, rather than relying on host-tool discovery. Record
the selected generated-asset digests and demonstrate a build with outbound
network denied after Buildroot has fetched the pinned source archive. This
audit is static only: neither `make`, an asset generator, Docker nor any
upstream installer has been executed.

### Production build-boundary findings (2026-10-27)

This is a source-level inspection of the candidate only; it is not a target
build or an approval of any dependency.

- `ad5m-br` is a useful upstream *platform setting*, not a complete Buildroot
  recipe.  It selects `fbdev` and `evdev`, and turns off SDL, GLES and the
  screensaver.  It leaves mocks enabled by the Makefile default and enables
  remote control unless `HELIX_PACKAGING=1` or an equivalent explicit setting
  is provided.  The package must set all of
  `PLATFORM_TARGET=ad5m-br`, `CROSS_COMPILE=$(TARGET_CROSS)`,
  `ENABLE_MOCKS=no`, `HELIX_PACKAGING=1`, and
  `ENABLE_REMOTE_CONTROL=no`; none may depend on a host default.
- The upstream `all` target also requests the optional Bluetooth plugin.  That
  component brings a BlueZ/systemd/minilzo path and is irrelevant to the
  initial AD5M UI.  The package must instead build only the reviewed runtime
  executables (`helix-screen`, splash and watchdog) and then use the upstream
  `install DESTDIR=...` staging target.  It must never invoke the installer,
  deploy or update targets.
- The package must not allow the normal target to auto-regenerate fonts based
  on a host `lv_font_conv` discovery. It needs a small reviewed adaptation that
  treats the pinned generated font files as inputs and disables the untracked
  `.fonts.stamp` recipe. Asset-regeneration, `npm`, Docker and updater targets
  remain developer-only and out of the Buildroot package path.
- With those production flags, desktop SDL, ALSA and libusb code paths are
  compiled out.  The app still needs LVGL/ThorVG, helix-xml, lv-markdown/md4c,
  quirc, GLM, fonts/translations, libhv, WPA client, libnl, spdlog/fmt, zlib,
  pthreads and optional OpenSSL.  `cpp-terminal`, FTXUI, OpenVDB, SDL2,
  linenoise and the Bluetooth plugin are not part of that initial target
  graph. The Git-submodule closure is now known, but vendored and generated
  inputs still need their own identity and license review.
- Current upstream cross-build logic builds libhv, WPA client and libnl from
  source submodules, probes spdlog/fmt with host-oriented paths, and adds
  `-L/usr/lib/$(TARGET_TRIPLE)`.  Buildroot's SDK provides neither that path
  nor a `CROSS_COMPILE` variable by default.  A small, separately reviewed
  Buildroot adaptation is therefore required before packaging: make the
  recipe pass `$(TARGET_CROSS)`, use the staging sysroot rather than host
  probes/paths, and decide each library's owner (a pinned audited submodule or
  a reviewed Buildroot package).  Do not add an "as-is" recipe that can link
  host artifacts accidentally.
- `ad5m-br` also compiles the PWM sound backend with automatic export of
  `/sys/class/pwm/pwmchip0/pwm6` when that channel is absent.  That writes to
  hardware state during UI initialization.  The first project image must
  compile sound/PWM support out (including the auto-export flag) until a
  separate AD5M hardware review establishes the channel, ownership, resource
  impact and shutdown behaviour.  M300 output likewise remains disabled until
  the printer configuration is explicitly reviewed; no UI package may add or
  modify a beeper macro.
- The pinned Buildroot tree already supports the normal source-delivery
  mechanism needed here: `SITE_METHOD=git` with `GIT_SUBMODULES=YES` checks
  out the root commit, recursively resolves its Gitlinks, and produces the
  package source archive.  A future recipe may use that mechanism with the
  immutable root commit; it must not use a branch, an installer download, or
  a custom fetcher.  Before enabling it, record each recursively fetched
  commit, URL, license and license-file location, then retain the resulting
  source-archive digest as build evidence.  This capability has been read
  from the canonical Buildroot downloader only; it has not been run for
  HelixScreen.

### Runtime-asset and staging audit (2026-10-27)

The checked-out root source excluding `.git` occupies about 340 MiB, largely
because `assets/test_gcodes` alone is about 179 MiB.  The upstream `install`
target does **not** copy those test G-codes, but it does unconditionally copy
all of `assets/fonts`, `assets/images`, `assets/sounds`, `assets/config`, and
`filaments.json`.  At this candidate revision that staged asset set is about
73 MiB: approximately 41 MiB of fonts and 32 MiB of images.  This is a
source-level size inventory, not a target-image measurement.

That install behaviour is too broad to accept blindly on the constrained
AD5M.  It also cannot be safely trimmed by filename guessing: production code
uses the installed CJK font directory, runtime printer-image fallbacks,
pre-rendered splash assets, configuration seeds, sounds, and the filament
catalog.  Before the package is enabled, create a reviewed AD5M runtime-asset
manifest that maps every staged file class to a code/config consumer and
measures the resulting image, RAM and startup cost.  The package may invoke
the upstream `install DESTDIR=...` only through a staging wrapper that removes
or avoids manifest-excluded files deterministically; it must never copy
upstream test data or use a hand-waved whole-asset-tree install.

The upstream root target also includes a Bluetooth-plugin target whose build
is conditional on detected libraries.  The first AD5M package must continue
to invoke only explicit reviewed runtime executables, and must make feature
selection independent of which libraries happen to exist in the build host or
SDK.

### Buildroot ownership audit (2026-10-27)

The pinned canonical Buildroot tree already packages and stages `libnl`,
`fmt`, `spdlog`, OpenSSL, zlib and JPEG.  Its `spdlog` package depends on its
external-`fmt` package, so it is a coherent SDK dependency pair rather than a
reason to use HelixScreen's host-path probes.  The existing AD5M SDK
configuration already selects OpenSSL, zlib and turbo JPEG; it does not select
`libnl`, `fmt`, or `spdlog`.

Buildroot also has a normal upstream `wpa_supplicant` package, but the current
AD5M configuration uses IWD instead.  HelixScreen's direct `hostap` Gitlink is
therefore not an acceptable automatic dependency: the first package must not
introduce a second Wi-Fi-management stack merely to satisfy an optional UI
feature.  First establish whether that feature can be compiled out or
integrated through the project's selected network manager; otherwise defer it.

The provisional ownership direction is: prefer Buildroot for `fmt`, `spdlog`,
`libnl`, OpenSSL, zlib and JPEG where the audited target graph needs them;
retain a pinned audited upstream source only where there is no suitable
Buildroot package (for example LVGL, libhv, helix-xml and lv-markdown).  This
is a dependency-selection candidate, not an enabled package configuration.

## Repositories affected

- ad5m_enhanced_mod
- ad5m-project

## Requirements

- Confirm or reject `v1.0.6` after reviewing its source graph, then pin the
  selected official release/tag and record the source identity, archive/Git
  integrity evidence, license, and release-selection rationale.
- Review every fetched direct submodule and external dependency for source
  identity, license, Buildroot ownership, and whether it can be replaced by an
  existing Buildroot package. The fixed candidate has no nested Git
  submodules; repeat the closure audit before accepting any changed Gitlink.
- Do not add a second Wi-Fi-management stack; any HelixScreen network-control
  path must be explicitly compiled out or integrated with the selected IWD
  design after a separate review.
- Keep the package in `ad5m_enhanced_mod` unless a dependency is demonstrably
  generic Buildroot functionality.
- Build the upstream `ad5m-br` target through the canonical Buildroot SDK;
  do not use upstream Docker toolchains or deployment targets.
- Do not enable the upstream PWM sound/auto-export or M300 paths in the first
  package.  Reconsider either only as a separately reviewed hardware feature.
- Use direct `fbdev` display output; do not add X11, Wayland, GTK, or a desktop
  stack.
- Install immutable application assets under `/opt/helixscreen`; persist
  `settings.json`, `helixscreen.env`, themes, and user layout under
  `/root/printer_data/config/helixscreen` through a package-owned service.
- Do not execute, ship, source, or patch in any upstream installer/deployer.
  Do not rely on Forge-X, ZMOD, Klipper Mod, or other community integration
  scripts.
- Provide an AD5M-specific SysV service that starts after the needed printer
  services without blocking their startup, handles a failed UI gracefully, and
  does not stop Klipper or Moonraker.
- Replace the `lite` welcome framebuffer only in the HelixScreen variant;
  retain `lite` as a minimal no-GUI image.
- Add software-only checks for package selection, service syntax, staged file
  ownership, and the absence of legacy-screen packages.  Keep physical display,
  touch, boot, and memory validation as separate acceptance evidence.

## Non-goals

- Running an installer, update script, or deploy command on any printer.
- Changing Klipper safety configuration, MCU firmware, heaters, motion,
  probing, or power control.
- Treating a host mock screenshot as target or hardware validation.
- Adding in-application self-update before a transactional project update
  design exists.

## Proposed implementation

1. Complete source/dependency closure for the `v1.0.6` candidate using
   AD5M-008's integrity standard; select another fixed source only with a
   recorded reason.
2. Add a project-specific Buildroot package with a small reviewed Buildroot
   adaptation that uses the SDK sysroot and invokes only the explicit runtime
   targets and staging interface, with an explicit, reviewed set of target
   library dependencies.
3. Add a separate `helixscreen` variant instead of changing `lite` in place.
4. Add a small project-owned SysV init script and fixed environment that sets
   `HELIX_DISPLAY_BACKEND=fbdev`, the persistent `HELIX_CONFIG_DIR`, and only
   hardware values verified on an AD5M.
5. Add an upgrade-safe seed/migration policy for default settings without
   overwriting user layout or connection settings.
6. Validate build/configuration and host mock separately; schedule physical
   testing only after an installable image exists.

## Acceptance criteria

- A reviewed immutable source and all recursively fetched source identities
  are recorded, including licenses and integrity checks.
- `make helixscreen_prepare` configures a separate no-legacy-screen variant.
- The target package builds from the canonical pinned Buildroot source without
  upstream Docker images or installers.
- The staged image has `/opt/helixscreen` assets and a writable persistent
  configuration directory, with no default setting that overwrites a user's
  dashboard layout on rebuild/upgrade.
- The staged runtime-asset manifest is reviewed and its image, RAM and startup
  impact is measured; no test G-code or unreviewed asset tree is installed.
- The service does not retain a competing UI and fails without destabilizing
  Klipper or Moonraker.
- Host mock, target build, and physical-printer evidence are reported as three
  distinct test layers.

## Verification

Upstream source inspection and Makefile variable evaluation only. The
inspection evaluated the `ad5m-br` linker flags with a supplied cross-prefix;
it did not compile, link or execute HelixScreen. A metadata-only Git closure
audit confirmed the fixed root commit and all 15 direct Gitlink commits, with
no nested Gitlinks at those revisions. Build files and the relevant generator
scripts were read statically to identify the font-generation and downloader
paths; pinned root-license texts were also read. None of those paths was
invoked. No package has been created, no source submodule has been accepted,
and no HelixScreen binary, installer, updater, or deployment script has been
run.

## Hardware verification

Not run.

## Remaining work

- Audit vendored code, generated inputs, build-time downloads and target
  libraries for the `v1.0.6` candidate; the fixed Git-submodule closure is
  recorded above and contains no nested Git source.
- Design and review the no-regeneration Buildroot adaptation, record the
  selected generated-asset digests, and demonstrate no outbound access after
  the pinned source archive has been fetched.
- Produce and review the minimal AD5M runtime-asset manifest before accepting
  the upstream staging target.
- Decide and validate the Buildroot-versus-pinned-source owner for every
  runtime dependency, including an explicit no-duplicate-Wi-Fi decision.
- Confirm or reject the candidate after that audit.
- Measure the packaged image's disk/RAM impact and boot behavior.
- Determine exact framebuffer, touch device, calibration, and backlight values
  from a physical AD5M before setting hardware-specific environment values.
