# 0001 Treat historical third-party sources as untrusted inputs

Status: accepted

## Context

Some historical AD5M sources can be useful for understanding hardware support
or identifying an approach, but they are not project upstreams and must not be
trusted as a source of executable code or build dependencies. In particular,
the historical Buildroot reference must not be used as an automatic build
dependency or imported wholesale.

## Decision

The project may inspect historical third-party repositories for ideas or for a
specific, identified change. It will not merge, rebase onto, bulk cherry-pick,
or otherwise import their history. No unreviewed dependency, submodule,
download URL, binary, credential, service, or build artifact from such a
source may enter the project.

Before adopting any individual change inspired by a historical source:

1. Identify the exact code or behavior being considered and its provenance.
2. Check official upstreams for an equivalent supported implementation.
3. Independently implement or port the minimum necessary change.
4. Perform a heightened security review, including installers and update
   paths, privilege boundaries, network access, credentials, remote downloads,
   service configuration, and opaque/generated artifacts.
5. Record the rationale, review, and software/hardware verification in the
   scoped task.

## Interpretation

This policy is deliberately strict because these sources can influence a
privileged embedded firmware build. It is not a ban on reading historical code,
learning from it, or porting a specific useful idea.

The purpose is to prevent an old third-party tree from silently becoming a
trusted upstream or executable supply-chain dependency. If this policy becomes
unnecessarily restrictive, change it explicitly through a later ADR rather
than bypassing it in an implementation task.

## Consequences

This approach may take longer than copying a historical implementation, but
keeps the supply chain auditable and preserves the official Buildroot project
as the authoritative upstream. Existing legacy references must be removed or
replaced through reviewed, scoped work; they must not be initialized or updated
as routine development setup.
