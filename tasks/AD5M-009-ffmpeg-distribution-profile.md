# AD5M-009 FFmpeg distribution profile

Status: proposed

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

Not run.

## Hardware verification

Not run.
