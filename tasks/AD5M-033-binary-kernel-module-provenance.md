# AD5M-033 Binary kernel-module provenance

Status: in progress

## Goal

Establish source, license, kernel/config compatibility and reproducible build
evidence for tracked binary kernel modules, or remove them from the release
profile through an explicitly tested replacement/removal decision.

## Context

AD5M-028 found tracked ARM relocatable modules under
`build_overlays/common/lib/modules/`: `cbc.ko` (SHA-256
`63e6f2f760c78e5a7a07bd8f39182b5c61ebfd302f8ad38d3b920dd2ca0e337d`) and
`md4.ko` (SHA-256
`aad89138467cb3d95fdc23d83fe9bc2298c0f1dc11cfd1b6a81a382f5bfbc156`), plus
`sha512_generic.ko` (SHA-256
`f3af01634be9b58db19edbeb7ca2018d443de6b133e94de798bc012e4fbb7833`).  The
inspected project tree has no adjacent source, build recipe, license file or
provenance record.  They may be legitimate compatibility modules; their
binary form alone is not evidence of harmful behavior.  It is nevertheless
not adequate for a release manifest or a source-review policy.

## Static findings — 2026-10-28

- All three are 32-bit ARM EABI5 relocatable ELFs and report `license=GPL`,
  `intree=Y`, and `vermagic=5.4.61 SMP preempt mod_unload ARMv7 p2v8`.
  Their embedded descriptions and aliases identify standard Linux crypto
  functionality: CBC, MD4, and generic SHA-512/SHA-384.  This is compatible
  with their being built from a Linux 5.4.61 source tree, but metadata is not
  cryptographic proof of their source or of a safe build configuration.
- Historical commit `a94d3ffa2257c06467e0a794376836cdd31c5f53` (2024-02-27,
  subject `Add kernel modules missing for wifi with iwd`) introduced all three
  binary blobs unchanged.  Their hashes match that initial commit.  Later
  commits only moved their paths with the overlay directory.
- `build_overlays/common/etc/init/init_chroot.sh` unconditionally loads all
  three before the vendor `8821cu.ko`, under a comment saying they are for
  Wi-Fi.  The modules declare no module dependencies.  The repository uses
  iwd for Wi-Fi, but this static evidence does not prove which authentication
  mode or driver consumes each module, nor that every module is required for
  the supported profile.
- This project builds a userspace SDK against 5.4 kernel headers; it does not
  contain or build the running vendor kernel.  Therefore the exact kernel
  `.config`, source tree, compiler and module ABI needed for a reproducible
  rebuild are currently unavailable here.  No module was loaded, disassembled,
  rebuilt, replaced or tested during this audit.

## Resolution path

1. Obtain the exact vendor kernel source and configuration corresponding to
   the installed 5.4.61 ABI, or a vendor stock artifact containing byte-equal
   modules with a documented provenance chain.  Record artifact hashes and
   license/source offer evidence.
2. Rebuild the three modules from that exact source/config/toolchain and
   compare hashes where build reproducibility permits.  If byte identity is
   not attainable, record the explained difference and compare ELF/module
   metadata before any target test.
3. Independently establish the minimal Wi-Fi/authentication scenario needing
   each module.  If a module can be eliminated, make that a separately
   reviewed, target-tested change; do not infer it from module names.
4. Until then, treat these as explicitly inventoried legacy blobs.  They are
   not eligible for a broadly distributed release, and a personal-alpha image
   needs an explicit acceptance decision in AD5M-031 rather than silently
   inheriting them.

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

Static audit completed on 2026-10-28: hashes, ELF module metadata, Git
introduction history and boot-time load site were inspected without loading
the modules.  The required source/config provenance and reproducible rebuild
evidence are still absent.

2026-10-29: the integration validation gained a deliberately narrow legacy
module inventory. It pins the three known overlay module filenames and
SHA-256 values, rejects an added, removed or changed overlay `.ko`, and checks
that the chroot boot script names fixed paths for those three modules rather
than constructing an overlay module path from a variable. This catches silent
binary drift and unreviewed expansion of the legacy blob set; it does **not**
establish source provenance, inspect behavior, prove ABI compatibility or make
the blobs acceptable for broad release. `make validate` passed on the host;
no module was loaded or target kernel used.

## Hardware verification

Not run.  Any final removal/replacement requires hardware validation of the
specific feature/driver consumer.
