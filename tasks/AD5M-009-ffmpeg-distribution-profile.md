# AD5M-009 FFmpeg distribution profile

Status: in progress

## Goal

Choose and verify an FFmpeg profile that matches the intended distribution
model without silently losing required camera/video behavior.

## Context

AD5M-007 found the current `base` fragment enables both
`BR2_PACKAGE_FFMPEG_GPL=y` and `BR2_PACKAGE_FFMPEG_NONFREE=y`, with
`BR2_PACKAGE_X264=y` and `libx264` in the requested encoder list. FFmpeg's
configure output therefore identifies the build as “nonfree and
unredistributable”.

`libx264` is GPL-compatible but makes the FFmpeg combination GPL when enabled;
it is not, by itself, the reason for the nonfree designation. The exact
component that requires `--enable-nonfree` must be identified from the
generated FFmpeg configuration before changing the profile.

## Candidate profiles

- **Local-only experimental build:** retain the current flags, label it
  nonredistributable, and never publish it as a release artifact. This is not
  a release solution.
- **Redistributable GPL profile:** remove `BR2_PACKAGE_FFMPEG_NONFREE`, retain
  GPL and `libx264` only if the project is prepared to meet GPL obligations for
  the combined distribution, and publish corresponding sources, modifications,
  build instructions, notices and license materials.
- **Redistributable LGPL-oriented profile:** remove both nonfree and GPL
  selections, remove `x264`/`libx264`, and keep only codecs/features that have
  an LGPL-compatible configuration. Existing H.264 recording/streaming needs
  a functional replacement or an explicit feature reduction.

## Requirements

- Determine the exact FFmpeg configure switches and linked libraries that
  trigger `--enable-nonfree`; do not infer this from the Kconfig name alone.
- Establish whether the camera stack needs H.264 encoding, H.264 decoding, or
  only MJPEG before selecting a profile.
- Build and inspect `ffmpeg -L`, `ffmpeg -buildconf`, enabled encoders and
  decoders in the resulting ARM rootfs.
- If publishing a GPL profile, prepare a release manifest and source-offer
  process that covers FFmpeg, x264 and all local patches. Obtain legal review
  before asserting compliance.
- If selecting the LGPL-oriented profile, run camera/streaming validation on
  hardware before releasing it.
- Keep the choice recorded in the release manifest; do not ship an artifact
  whose distribution status is ambiguous.

## Non-goals

- Changing unrelated FFmpeg versions or codecs.
- Treating build success as camera validation.
- Giving legal advice or declaring compliance without review.

## Verification

2026-10-27 software-only checks passed:

- `make -C ad5m_enhanced_mod validate`
- `make -C ad5m_enhanced_mod validate-config`

The latter created and removed a disposable Buildroot output directory.  It
confirmed the canonical Buildroot revision and generated a lite configuration
with the nonfree FFmpeg feature disabled while GPL and x264 remained selected.
No FFmpeg package or target image was compiled.

## Hardware verification

Not run.

## Implementation notes

### 2026-10-27 configuration and camera-consumer audit

The exact cause of the prior nonredistributable result is now identified.  In
the pinned Buildroot `ffmpeg.mk`, `BR2_PACKAGE_FFMPEG_NONFREE=y` unconditionally
adds `--enable-nonfree`; FFmpeg then records `CONFIG_NONFREE=1`, even if no
nonfree optional codec is selected.  The previously generated AD5M FFmpeg
configuration confirms that it passed both `--enable-nonfree` and
`--disable-libfdk-aac`, while `CONFIG_LIBFDK_AAC=0` and `CONFIG_OPENSSL=0`.
The nonfree status was therefore caused by the project fragment's broad
permission itself, not by `libx264` or a detected FDK-AAC/OpenSSL dependency.

The camera listener is independent of FFmpeg: the AD5M overlay launches
ustreamer with an MJPEG-capable V4L2 device and `-m MJPEG`.  FFmpeg is used by
the Moonraker timelapse extension, whose rendering command explicitly needs
the `libx264` encoder to create an MP4 file; its selected MJPEG decoder,
`image2` demuxer, MP4 muxer and scaling/rotation filters support that path.
It does not need H.264 decoding for the observed camera or timelapse workflow.

The base fragment consequently selects the redistributable GPL configuration:
it retains `BR2_PACKAGE_FFMPEG_GPL=y`, `BR2_PACKAGE_X264=y` and the constrained
timelapse codec list, while explicitly disabling
`BR2_PACKAGE_FFMPEG_NONFREE`.  The Buildroot-config validation now rejects a
generated lite configuration that re-enables nonfree FFmpeg or loses the
reviewed GPL/x264 timelapse profile.  This is a source/configuration decision,
not a declaration of complete GPL-compliance or a release approval.

Before publication, build an image from the changed profile and inspect
`ffmpeg -buildconf`, `ffmpeg -L`, the selected encoder/decoder list and the
target dependency/license manifest.  Then render a representative timelapse
and test the USB MJPEG camera on hardware.  A release still needs a reviewed
source-offer, notices and licensing process for FFmpeg, x264 and all project
modifications.

### Packaging and optional-media direction — 2026-10-28

Read-only inspection of the existing local target artifact shows that the
configured `/usr/bin/ffmpeg` is a stripped ARM executable of 296,452 bytes.
Its selected `libav*`, `libsw*` and `libx264` files occupy about 4.2 MiB in
that target tree; this is a useful sizing observation, not a measured clean
image delta because some shared libraries may also serve other packages.

FFmpeg is **not** required for the USB camera's live MJPEG stream: that path
is ustreamer/V4L2.  The project-owned timelapse component checks for
`/usr/bin/ffmpeg`; when absent it disables automatic rendering and reports a
render error instead of trying to fetch software.  When present, it invokes a
constrained local command to turn saved JPEG frames into an MP4 with
`libx264`.  Existing documentation also warns that rendering or other heavy
background work while printing can cause print failure on the constrained
AD5M.

Selected direction for the personal alpha: retain the reviewed, constrained
FFmpeg profile in the built image while timelapse rendering remains a
supported feature.  The printer must never self-download FFmpeg or any other
executable at runtime: that would bypass source provenance, package integrity,
rollback and storage-budget controls.  Nor should it execute an arbitrary
binary copied from USB.

If the 4 MiB-class media footprint later proves undesirable, offer a distinct
optional **media/timelapse add-on** through the project's normal update
mechanism.  It must be an exact AD5M artifact with a manifest, compatible
component versions, integrity/authenticity verification, atomic
install/rollback and a resource policy that prevents rendering during a
print.  Offline USB delivery may carry that same verified project artifact;
USB is a transport, not an authority to run whatever executable it contains.
This option is deferred until AD5M-019/025/027 provide the required update
and recovery contracts.
