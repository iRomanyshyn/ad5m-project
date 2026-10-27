# AD5M-023 HelixScreen local-only profile viability

Status: proposed

## Goal

Determine whether an upstream-pinned HelixScreen revision can provide a small,
auditable compile-time local-only profile suitable for the AD5M package.

## Context

AD5M-018 rejected HelixScreen `v1.0.6` as the first production candidate.  Its
top-level source list compiles updater, telemetry, crash-report and debug-
bundle components by default; they have direct application/UI call sites and
no single feature gate comparable to `ENABLE_REMOTE_CONTROL=no`.  A broad
private fork that replaces many components with stubs would enlarge the
trusted code and audit surface, contrary to the project goal of a minimal
maintainable integration.

This is not a claim that upstream is unsafe or malicious.  It is an AD5M
product requirement: local-only must be demonstrable from the built artifact,
not inferred from hidden UI controls or mutable runtime settings.

## Requirements

- Prefer a later official upstream revision that offers documented,
  compile-time feature selection for all third-party update, telemetry, crash
  and debug-bundle transport.
- A local-only build must exclude both transport implementation and all
  associated endpoint/credential material; it must not merely disable a UI
  control or postpone a request with an environment variable.
- Preserve local Moonraker functionality while excluding remote-control
  server/client, upstream self-update, crash upload and debug-bundle upload.
- Keep telemetry entirely absent in the first package.  A future contribution
  feature remains separately opt-in under AD5M-018 and may not reuse the broad
  upstream identity/queue/event model.
- Keep the change set small enough for line-by-line source, license and
  no-egress review; do not introduce an unbounded maintained fork.

## Candidate acceptance evidence

- A pinned official source revision and full source/submodule closure audit.
- An explicit feature matrix showing which source files and UI paths are
  compiled in each profile.
- Static artifact scans prove the local-only binary/files lack known update,
  telemetry, crash and debug-bundle hosts and credentials.
- A no-egress software test permits only a local Moonraker fixture and proves
  no DNS/HTTP request is attempted for an idle local-only session.
- Host mock, target build and hardware evidence are labelled separately.

## Non-goals

- Re-enable upstream self-update, diagnostics upload, remote control or
  telemetry merely to make a build pass.
- Run an installer, deploy command, or arbitrary upstream script on a printer.
- Claim that a static scan alone validates fbdev, touch, memory or printer
  safety.
