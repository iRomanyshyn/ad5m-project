# AD5M-007 Canonical package-build validation

Status: complete

## Goal

Build and inspect one or more Klipper Mod package variants using the canonical
Buildroot source contract established by AD5M-001, before considering any
printer installation.

## Context

AD5M-001 verified the external-tree configuration for SDK, lite,
KlipperScreen, and GuppyScreen variants, but deliberately did not download
package sources or compile firmware. A full build is required to validate the
source package set, toolchain, artifact creation, and disk-space assumptions.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod
- buildroot-flashforge-ad5m

## Requirements

- Start from the compatible commit pair recorded in AD5M-001, or explicitly
  record any reviewed successor pair.
- Confirm required disk space and network/source-download policy before
  beginning.
- Build the smallest useful variant first; do not assume all variants are
  equivalent.
- Record every external source downloaded, hash/commit pinning available, and
  any source or license issue discovered.
- Inspect package names, checksums, rootfs/chroot contents, and logs.
- Keep generated output local and out of Git.

## Non-goals

- Installing a package on a printer.
- Claiming motion, heater, MCU, display, or print validation.
- Updating package versions merely to make the build succeed.

## Proposed implementation

Measure available disk space, run the lite variant first using the canonical
contract, then inspect resulting output and build logs. Decide whether the UI
variants warrant separate builds based on their distinct dependencies.

## Acceptance criteria

- The selected variant completes or a precise reproducible failure is recorded.
- Generated artifacts can be identified and their integrity data inspected.
- The exact source commit pair and commands are recorded.
- No result is represented as a physical-printer test.

## Implementation notes

### Inputs and command

The validation started from the compatible pair recorded by AD5M-001:

- Klipper Mod: `b33f8cc9d8c1955a542caccaec31983d786d6da3`
- canonical Buildroot: `77fb3a9ec7cd1e27910a441e0d75df0cbe51c2de`

There was 328 GiB free before the build. The selected first variant was
`lite`; the command was `make -C ad5m_enhanced_mod lite`.

The first invocation successfully built the SDK, then exposed a stale external
toolchain path in the generated variant config. `base` previously kept a
relative path that Buildroot resolved below its own source tree, rather than
below the Klipper Mod output tree. The wrapper now removes that fragment value
and writes the absolute `BUILDROOT_SDK/host` path into each generated variant
config. The corrected config contains:

```text
BR2_TOOLCHAIN_EXTERNAL_PATH="/home/iromanyshyn/Projects/ad5m/ad5m_enhanced_mod/build_output/buildroot/sdk/host"
```

The second invocation passed `toolchain-external-custom`, compiled and
installed the SDK and a substantial portion of the `lite` rootfs, then stopped
before rootfs/package creation at this reproducible error:

```text
make[2]: *** No rule to make target 'python-msgspec', needed by
'.../build/klipper-v0.13.0-285-gfc2aaae69-ffadm5/.stamp_configured'. Stop.
* Error: variant-lite: build failed
```

`build_scripts/buildroot/package/klipper/Config.in` selects
`BR2_PACKAGE_PYTHON_MSGSPEC`, and `klipper.mk` requires `python-msgspec`, but
the pinned canonical Buildroot tree contains neither that Kconfig symbol nor a
`python-msgspec` package. Git history shows the selection originated in an
historical Klipper-package commit that referenced an untrusted source. This
observation is evidence only: no code or package was fetched or imported from
that source. Removing the dependency solely to make the build pass would be
unsafe because the exact Klipper revision has not yet been audited to prove it
is unnecessary. The follow-up is AD5M-008.

### Sources and integrity observations

Buildroot's `dl/` cache contains the following sources downloaded or reused by
this attempted SDK plus `lite` build (directory names are the package source
record):

```text
acl arm-gnu-toolchain attr autoconf autoconf-archive automake binutils bison
blake3 busybox ca-certificates ccache cmake dbus dropbear ell expat fakeroot
ffmpeg file flex fluidd gawk gcc gettext-tiny git glibc gmp gnutls hiredis
hwdata iproute2 iptables iwd jpeg-turbo keyutils libaio libbsd libcamera
libcurl libevent libffi libgpiod libmd libopenssl libpcap libpng libsodium
libtasn1 libtirpc libtool libunistring libuv libv4l libxcrypt libyaml libzlib
linux lshw lvm2 m4 meson mpc mpdecimal mpfr nano ncurses netcat-openbsd
nettle nfs-utils ninja openresolv patchelf pkgconf popt python-calver
python-can python-cffi python-cython python-dbus-fast python-dbus-next
python-distro python-flit-core python-greenlet python-hatchling python-ifaddr
python-installer python-jinja2 python-markupsafe python-msgpack python-numpy
python-packaging python-paho-mqtt python-pathspec python-periphery
python-pillow python-pip python-pluggy python-ply python-poetry-core
python-pycparser python-pynacl python-pypa-build python-pyproject-hooks
python-pyyaml python-serial python-setuptools python-setuptools-scm
python-tornado python-trove-classifiers python-typing-extensions python-uvloop
python-wheel python-wrapt python-zeroconf python3 readline rsync screen socat
sqlite tcpdump time tzdata ustreamer util-linux vim wireless_tools x264 xxhash
xz zic zstd
```

Buildroot reported its configured checksum verification for archive packages
with hash records (for example, the downloaded Arm GNU Toolchain 13.2.rel1
was verified as SHA-256
`6cd1bbc1d9ae57312bcd169ae283153a9572bd6a8e4eeae2fedfbc33b115fdbb`).
The package cache itself must remain local generated state, not a release
manifest.

Project-specific source declarations inspected in this run are:

- Klipper: `xblax/klipper`, `v0.13.0-285-gfc2aaae69-ffadm5` (not fetched,
  because its missing dependency blocked it);
- Moonraker: `Arksine/moonraker`, `v0.9.3`;
- Fluidd: release asset `v1.34.4`, downloaded as `fluidd.zip`, local
  SHA-256 `582d5f32cbb5bdda9bbfe0dc39aeb251457e6595a9331e30d9c8c94dcbbf0588`;
- backlight: `consp/flashforge_ad5m_backlight`, fixed commit
  `3cd9a0898e51a18e83976c7cab29037f8364f406`;
- audio: PyPI source `ff-ad5m-audio-0.4.tar.gz`, recipe SHA-256
  `9012c1fa3a6638f2092f30a131bd43370f3351bbb55835f6bdeff17dfff96d7e`;
- KlipperScreen: fixed commit `6b6f63b6639e5b85a2b3f4b0b0cd7677ac9cda3d`;
- Klippain ShakeTune: fixed commit
  `c8ef451ec4153af492193ac31ed7eea6a52dbe4e`;
- GuppyScreen: mutable `origin/flashforge_ad5m` reference.

Two release-integrity findings require follow-up: the Fluidd recipe has no
tracked hash file (Buildroot explicitly warned `no hash file for fluidd.zip`),
and GuppyScreen uses a mutable branch reference. Neither was changed during
this validation run.

The existing FFmpeg selection includes `BR2_PACKAGE_FFMPEG_NONFREE=y` and
`BR2_PACKAGE_X264=y`; its configure output labels the result “nonfree and
unredistributable”. A generated image must not be represented as a freely
redistributable release without a separate licensing decision.

The build also showed ignored upstream subtarget failures while continuing:

- Nettle emitted an ignored `libhogweed.so` subtarget failure.
- LVM2 attempted to execute an ARM manpage generator on the x86 host and
  ignored the resulting `Exec format error`.

Those were not treated as proof of a healthy final image; no final image was
produced.

### Partial-output inspection

At failure, `variant-lite/target` exists and is 283 MiB; it contains Fluidd
under `/opt/fluidd` and an ARM EABI5 BusyBox whose interpreter is
`/lib/ld-linux-armhf.so.3`. The build directory is 5.2 GiB. `images/` is
empty, no rootfs tarball/package/checksum was generated, and no chroot smoke
test was attempted. `qemu-arm` was not available. The target directory carries
Buildroot's `THIS_IS_NOT_YOUR_ROOT_FILESYSTEM` marker, so it was inspected only
as a build staging tree and never run or installed.

## Verification

- `bash -n ad5m_enhanced_mod/build_scripts/build.sh`
- `make -C ad5m_enhanced_mod lite` (twice: first exposed and then confirmed
  the toolchain-path correction; second run reached the precise
  `python-msgspec` closure failure above)
- inspected the generated variant `.config`, Buildroot contract SHA, build
  stamps/log, cache source list, partial target tree, Fluidd archive SHA-256,
  and ARM BusyBox ELF metadata.

## Hardware verification

Not run. No rootfs package was installed, no printer was connected or changed,
and no motion, heater, MCU, display, camera, or print behavior was tested.

## Remaining work

Resolve the reviewed `python-msgspec` package closure and source-integrity
findings in `tasks/AD5M-008-package-source-closure-and-integrity.md`, then
repeat the `lite` build from clean reviewed inputs before considering a UI
variant or printer installation.
