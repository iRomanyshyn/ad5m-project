# AD5M-033 Binary kernel-module provenance

Status: proposed

## Goal

Establish source, license, kernel/config compatibility and reproducible build
evidence for tracked binary kernel modules, or remove them from the release
profile through an explicitly tested replacement/removal decision.

## Context

AD5M-028 found tracked ARM relocatable modules under
`build_overlays/common/lib/modules/`: `cbc.ko` (SHA-256
`63e6f2f760c78e5a7a07bd8f39182b5c61ebfd302f8ad38d3b920dd2ca0e337d`) and
`md4.ko` (SHA-256
`aad89138467cb3d95fdc23d83fe9bc2298c0f1dc11cfd1b6a81a382f5bfbc156`).
The inspected project tree has no adjacent source, build recipe, license file
or provenance record.  They may be legitimate compatibility modules; their
binary form alone is not evidence of harmful behavior.  It is nevertheless
not adequate for a release manifest or a source-review policy.

## Requirements

- Identify the exact source tree, kernel release/config, compiler/toolchain,
  license and intended runtime consumer for every tracked module.
- Rebuild byte-identically where practical, or document a justified
  reproducibility limitation with strong artifact identity and review
  evidence.
- Determine whether the selected canonical Buildroot kernel can provide the
  needed functionality from configured source instead.  Do not replace or
  remove a module without a compatibility plan and target test.
- Include module identity in the release manifest and reject unexplained
  binary additions through a static release-inventory check.
- Keep all module loading bounded to known paths; do not add a mechanism that
  loads modules from removable media or update payload metadata.

## Verification

Not run.  Recorded hashes are source-tree observations only; no module was
loaded, reverse engineered, rebuilt or tested.

## Hardware verification

Not run.  Any final removal/replacement requires hardware validation of the
specific feature/driver consumer.
