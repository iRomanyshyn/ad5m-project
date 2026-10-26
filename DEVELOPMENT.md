# AD5M development workstation

This document recreates the three-repository AD5M development checkout on a
Fedora workstation. The canonical root is `~/Projects/ad5m`; do not rename the
sibling repositories.

## Scope and safety

The target is a resource-constrained printer and changes involving heaters,
motion, probing, MCU firmware, boot, or power management need separate
software and physical-printer verification. Do not build Buildroot as root.
Build output is local and must never be committed or copied between workstations
as a substitute for a reproducible build.

## Host prerequisites

Buildroot's `docs/manual/prerequisite.adoc` in the pinned base repository is
the source of truth when requirements change. On Fedora, install this suitable
starting set:

```bash
sudo dnf install \
    git \
    openssh-clients \
    gcc \
    gcc-c++ \
    binutils \
    make \
    patch \
    diffutils \
    findutils \
    which \
    sed \
    gawk \
    bc \
    bash \
    gzip \
    bzip2 \
    xz \
    tar \
    cpio \
    unzip \
    rsync \
    file \
    perl \
    wget \
    curl \
    python3 \
    ncurses-devel
```

Check GitHub SSH access before cloning:

```bash
ssh -T git@github.com
```

If no suitable SSH key is present, stop here and arrange GitHub access. Do not
generate, replace, or upload credentials without the account owner's approval.

## Clone the project

```bash
mkdir -p ~/Projects/ad5m
cd ~/Projects/ad5m

git clone git@github.com:iRomanyshyn/ad5m-project.git ad5m-project
git clone git@github.com:iRomanyshyn/ad5m_enhanced_mod.git ad5m_enhanced_mod
git clone git@github.com:iRomanyshyn/buildroot-flashforge-ad5m.git \
    buildroot-flashforge-ad5m

git -C buildroot-flashforge-ad5m switch ad5m-2025.02
```

The resulting layout must be:

```text
~/Projects/ad5m/
├── ad5m-project/
├── ad5m_enhanced_mod/
└── buildroot-flashforge-ad5m/
```

## Configure and verify remotes

The Klipper Mod fork must have our fork as `origin` and xblax as a fetch-only
reference remote:

```bash
cd ~/Projects/ad5m/ad5m_enhanced_mod
git remote -v
```

If `upstream` is missing, add and make it fetch-only:

```bash
git remote add upstream https://github.com/xblax/flashforge_ad5m_klipper_mod.git
git remote set-url --push upstream DISABLED
```

The Buildroot repository must retain the official Buildroot project as its
authoritative `upstream`; xblax is a historical fetch-only reference:

```bash
cd ~/Projects/ad5m/buildroot-flashforge-ad5m
git remote -v
```

If a reference remote is missing, add only that remote using the corresponding
command below. Then make both fetch-only and retrieve metadata:

```bash
# Run the applicable command(s) only when that remote is missing.
git remote add upstream https://gitlab.com/buildroot.org/buildroot.git
git remote add xblax https://github.com/xblax/flashforge_adm5_buildroot_chroot.git

git remote set-url --push upstream DISABLED
git remote set-url --push xblax DISABLED
git fetch upstream --tags
git fetch xblax
```

Do not merge either reference remote merely because it has newer commits. The
maintained branch is `ad5m-2025.02`, initially based on official Buildroot tag
`2025.02.19`.

Verify the complete checkout:

```bash
for repo in \
    ~/Projects/ad5m/ad5m-project \
    ~/Projects/ad5m/ad5m_enhanced_mod \
    ~/Projects/ad5m/buildroot-flashforge-ad5m
do
    echo
    echo "===== $repo ====="
    git -C "$repo" status --short --branch
    git -C "$repo" remote -v
    git -C "$repo" log -1 --oneline --decorate
done
```

A workstation is ready only when all three repositories exist, expected
remotes are present, the Buildroot checkout is on `ad5m-2025.02`, and there
are no unexplained modifications.

## VS Code and Codex

Ensure the VS Code command-line launcher is available:

```bash
code --version
```

Install the OpenAI Codex/ChatGPT extension through VS Code before starting
agent-assisted work. The editor extension is a development convenience, not a
build dependency: firmware builds must remain reproducible without depending on
a particular editor or extension version.

When troubleshooting editor or agent regressions, record the locally working
VS Code and Codex extension versions in an ignored local note rather than
hard-coding them into the project unless the project genuinely requires a
specific version.

Useful local checks include:

```bash
code --version
code --list-extensions --show-versions | grep -E '^openai\\.'
```

The tracked workspace uses relative sibling paths and can be opened from any
checkout location:

```bash
code ~/Projects/ad5m/ad5m-project/ad5m.code-workspace
```

If opening the workspace makes VS Code unresponsive, first distinguish an
editor/extension regression from a repository or workspace problem. Do not
modify project source merely to work around an editor-extension failure.

## Git identity and public commit privacy

These repositories are public. Before making commits on a new workstation,
check the configured author identity:

```bash
git config --global user.name
git config --global user.email
```

If the personal email address should not be published in commit metadata,
configure the GitHub-provided `noreply` address before creating new commits.
Do not rewrite existing published history solely for this reason unless the
user explicitly chooses to do so.

Verify the effective identity in a repository with:

```bash
git config user.name
git config user.email
```

## Current build status

The Klipper Mod Makefile provides targets such as `make help`, `make lite`,
`make klipperscreen`, and `make guppyscreen`. Its generated output is under
`ad5m_enhanced_mod/build_output/`, which is ignored and local-only.

At the audited revision, these Makefile targets source Buildroot from the
`submodules/buildroot` gitlink. That submodule is still configured to the
historical `loss-and-quick/buildroot-flashforge-5m` repository. The canonical
`../buildroot-flashforge-ad5m` checkout is intentionally not substituted by
this setup documentation, because doing so requires a reviewed, scoped change
to the build integration. Do not initialize or update that legacy submodule as
part of routine workstation setup. This is a reproducibility gap to resolve in
a dedicated task; it must establish a reviewed relationship to the official
Buildroot-derived repository and record the exact compatible revisions.

Until that task is completed, `make help` is safe for discovering declared
targets, but a produced package must not be described as reproducible from the
three canonical repositories alone. A future build task should document the
canonical build and variant commands, expected artifacts, storage needs,
clean/rebuild procedure, hashes, and the distinction between software-only and
printer testing.

## Working across computers

Git is the synchronization mechanism. Before leaving a workstation, inspect
each repository, record task state in `tasks/` when applicable, commit and
push only intended work, and confirm no important untracked file is left
behind. Do not automatically commit unfinished or sensitive material.

On another workstation, inspect status first, then fetch and compare branches:

```bash
git fetch --all --prune
git status --short --branch
git branch -vv
```

Do not blindly run `git pull` across all repositories. Local-only notes belong
in ignored `LOCAL.md` or `*.local.md` files; never commit credentials, printer
passwords, private keys, tokens, or machine-specific test credentials.
