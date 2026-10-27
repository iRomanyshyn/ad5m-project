# AD5M-023 HelixScreen local-only profile viability

Status: in progress

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

## Upstream candidate screen

### `v1.1.0-beta.13` (2026-10-27)

The official `prestonbrown/helixscreen` remote advertises the annotated
`v1.1.0-beta.13` tag, which resolves to root commit
`1649a473a76cdc37447f40b53351d6dbeceb7fab`.  This is a prerelease, not an
automatic replacement for the previously inspected stable candidate.  It was
examined as a root-only temporary checkout: no submodules were initialised,
and no build, test, installer, updater, deploy command or executable was run.
`git fsck --no-dangling` passed for that checkout.  This is a narrow source
screen, not the full source/submodule closure audit required for acceptance.

There is a positive upstream improvement, but it covers only one part of the
boundary.  `HELIX_PACKAGING=1` defaults `ENABLE_REMOTE_CONTROL` to `no`, and
the Makefile removes `src/remote/*.cpp` from `APP_SRCS` in that case.  An AD5M
package could use that mechanism only by passing both values explicitly and
verifying the resulting source list/artifact.

The root application source list otherwise remains a recursive wildcard over
`src/`, with filters for test/tool/Bluetooth, optional rendering/screensaver,
mocks and remote control.  The screen found no documented local-only,
telemetry, updater, crash-report or debug-bundle source-selection gate.
Consequently it still includes `system/update_checker.cpp`,
`system/telemetry_manager.cpp`, `system/crash_reporter.cpp` and
`system/debug_bundle_collector.cpp`, together with their lifecycle and UI
callers.  Static call-site inspection found that normal startup initializes
the updater, crash reporter and telemetry manager, and subject initialization
registers the print-outcome telemetry observer.

`HELIX_DISABLE_AUTO_UPDATES` remains an environment-controlled updater policy,
not a compilation boundary.  The checked source contains update endpoints for
GitHub and `releases.helixscreen.org`, the telemetry endpoint, and the crash
and debug-bundle endpoint.  The new diagnostic-upload gate improves the
default for a self-built artifact: its compile default is off unless an
official packaging marker is set.  However, the transport implementation and
endpoint strings are still compiled, and `HELIX_DIAGNOSTIC_UPLOADS=1` can
override that default at runtime.  It therefore does not meet the requirement
that local-only artifacts omit the transport and associated endpoint material.

This tagged beta does not advance to dependency closure, Buildroot packaging,
host mock, target build or hardware testing.  It is not eligible for the
first AD5M local-only production package.  The conclusion is limited to the
specified source boundary; it makes no claim that upstream has made a request
or that its code is malicious.  A later official source can be re-screened if
it adds narrow, documented compile-time gates for every prohibited transport.
