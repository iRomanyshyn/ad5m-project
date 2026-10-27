# AD5M-018 HelixScreen external-egress and update boundary

Status: proposed

## Goal

Ensure the AD5M HelixScreen variant is local-only by default: it may talk to
the configured local Moonraker service, but it must not contact upstream
HelixScreen, GitHub, a telemetry service, a crash service, or any other
third-party endpoint without a separately approved product feature and an
explicit user action.

## Context

AD5M-015's pinned upstream candidate is not a passive framebuffer client. A
static source audit at `v1.0.6`
(`11230009fa09b4509d0d05f482feaeef2e94c854`) established the following:

- `HELIX_PACKAGING=1` makes `ENABLE_REMOTE_CONTROL` default to `no`, and the
  Makefile then removes `src/remote/*.cpp` from `APP_SRCS`. The project must
  continue to pass both values explicitly and verify the result.
- UpdateChecker is separate from that switch. It is initialized for the main
  app and starts an automatic network check 15 seconds after Moonraker connects,
  then every 24 hours. The stable path queries HelixScreen R2/GitHub release
  endpoints and supports downloading and applying an upstream self-update.
  `HELIX_DISABLE_AUTO_UPDATES=1` makes the upstream regard the device as
  firmware-managed and suppresses both checks and the updater UI, but this is
  an environment policy, not a compile-time removal.
- Telemetry is compiled independently. Its default setting is false, but an
  opt-in starts DNS/HTTPS requests to
  `https://telemetry.helixscreen.org/v1/events` and posts a device event
  stream using an upstream ingest credential embedded in the binary.
- Crash reporting is independent of telemetry. The normal crash modal offers
  a user-initiated POST to `https://crash.helixscreen.org/v1/report`; the
  debug-bundle flow similarly uploads compressed diagnostics to
  `https://crash.helixscreen.org/v1/debug-bundle`. Those flows can include
  printer and log context, so they are not acceptable merely because they are
  user-visible upstream features.

This is an upstream-source observation, not a claim that any request has
occurred on an AD5M. No HelixScreen code, installer, updater or printer was
run during the audit.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Treat outbound network access as a product capability, not as a side effect
  of a UI package. The initial release has no approved third-party egress.
- Keep the local Moonraker WebSocket/HTTP connection functional. A prohibition
  on third-party egress must not accidentally block same-device printer control.
- Build with `HELIX_PACKAGING=1`, `ENABLE_REMOTE_CONTROL=no` and
  `ENABLE_MOCKS=no` explicitly; prove the remote-control server/client source
  is absent from the staged binary.
- Do not ship or rely solely on a mutable setting that happens to default
  telemetry off. Disable its navigation, collection and transport through a
  reviewed package adaptation, or remove the entire telemetry path at build
  time. The selected design must leave no embedded third-party telemetry
  endpoint or ingest credential in shipped executable assets.
- Disable the upstream update checker and self-updater as a project-owned,
  upgrade-safe policy. `HELIX_DISABLE_AUTO_UPDATES=1` in a root-owned service
  environment may be a short-term defense-in-depth measure, but it is not by
  itself sufficient evidence that future upstream code cannot egress. The
  package needs a reviewed compile/staging adaptation or a testable upstream
  build switch that removes the automatic check and update UI.
- Do not ship upstream crash-report or debug-bundle upload actions in the
  local-only variant. Preserve local crash evidence and offer an export path
  only after reviewing data redaction, ownership and user interaction; never
  auto-upload it.
- Do not embed or redistribute upstream ingest credentials when their network
  path is removed. Treat their presence in a staged binary as a failed package
  check, even if they are described upstream as write-only.
- A future opt-in external-support, telemetry or update feature requires its
  own ADR, privacy/data inventory, explicit consent and revocation design,
  endpoint ownership review, failure behavior, and software-only evidence.
  It must not silently re-enable an upstream service.

## Non-goals

- Blocking local Moonraker access, user-selected LAN printer access, or a
  separately designed project update mechanism.
- Operating an AD5M, contacting an upstream telemetry/crash service, or
  attempting an upstream self-update.
- Promising anonymous diagnostics without a data inventory and review.

## Proposed implementation

1. Make a minimal, auditable Buildroot-only HelixScreen adaptation with an
   explicit local-only feature gate. It must be pinned with the HelixScreen
   source and apply cleanly only to the reviewed commit.
2. Set a root-owned, fixed service environment as a second layer: suppress
   upstream updates, pin the local Moonraker URL, and do not inherit arbitrary
   environment variables from an interactive shell.
3. Remove or compile out the upstream external update, telemetry, crash-upload
   and debug-bundle-upload entry points. Retain only local crash files and
   local export after its separate review.
4. Add package checks that reject the known external endpoint hostnames and
   upstream ingest credential in all staged runtime executables. Check the
   generated service environment and selected source list too.
5. Exercise the staged binary against a local mock Moonraker endpoint while
   outbound connections are denied. Record local connection, idle operation,
   a disabled-update state and a disabled telemetry state separately from any
   physical-printer test.

## Acceptance criteria

- The staged application neither contains nor can invoke the upstream
  remote-control server/client, updater, telemetry transport, crash uploader
  or debug-bundle uploader.
- Static package checks reject `telemetry.helixscreen.org`,
  `crash.helixscreen.org`, GitHub release endpoints and the upstream ingest
  credential in shipped runtime files.
- The generated service environment is root-owned, explicit and does not rely
  on user-editable defaults to keep external egress off.
- A software-only test with outbound access denied proves that the UI can
  connect to a local mock Moonraker service and stay operational without an
  update/telemetry request.
- Local crash evidence survives for controlled export; no crash or diagnostic
  data leaves the device automatically or through an upstream upload action.
- Any later external-support feature has an approved ADR and its own tests.

## Verification

Pinned-source static audit only. The Makefile, app initialization, update,
telemetry, crash-report and debug-bundle source paths were read without
checkout, compilation or execution. No package, service or binary exists yet.

## Hardware verification

Not run.

## Remaining work

- Decide whether a suitable upstream compile-time local-only switch exists;
  otherwise design the smallest reviewable project adaptation.
- Inventory all staged executable and non-executable runtime files before
  choosing the endpoint/credential scan.
- Define a no-egress test harness that still permits an explicit local
  Moonraker fixture.
