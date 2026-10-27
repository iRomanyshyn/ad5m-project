# AD5M-018 HelixScreen external-egress and update boundary

Status: in progress

## Goal

Ensure the AD5M HelixScreen variant is local-only by default: it may talk to
the configured local Moonraker service, but it must not contact upstream
HelixScreen, GitHub, a telemetry service, a crash service, or any other
third-party endpoint. The sole candidate exception is an explicitly enabled
upstream-HelixScreen telemetry contribution feature; it needs the consent,
data and revocation controls below. It must not carry update, crash-report or
debug-bundle upload authority with it.

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

The candidate has one relevant build-time gate only: remote control. A static
search of its Makefile, `mk/` files, `include/` and `src/` found no equivalent
compile-time local-only switch for UpdateChecker, telemetry, crash reporting,
or debug-bundle upload. `HELIX_DISABLE_AUTO_UPDATES` is a runtime environment
policy for the updater alone. Therefore the first AD5M local-only variant
needs a small project adaptation unless a later pinned upstream revision adds
and documents suitable feature gates.

This is an upstream-source observation, not a claim that any request has
occurred on an AD5M. No HelixScreen code, installer, updater or printer was
run during the audit.

### Candidate compile-gate confirmation (2026-10-27)

A fresh temporary read-only clone of the official annotated `v1.0.6` tag was
resolved by the official remote to
`11230009fa09b4509d0d05f482feaeef2e94c854`. Static inspection of *that exact
root source* confirms that `HELIX_PACKAGING=1` only changes the default for
`ENABLE_REMOTE_CONTROL` and removes `src/remote/*.cpp`. Its Makefile has no
`ENABLE_DIAGNOSTIC_UPLOADS`, local-only, telemetry, crash-upload or
debug-bundle-upload compilation gate. A separately available local checkout
with a different history is not evidence for this pinned candidate and was
not used to select its behavior.

At the pinned source, `CrashReporter::try_auto_send()` directly POSTs to the
crash-worker URL, `DebugBundleCollector::upload_async()` directly POSTs a
compressed bundle to the debug-bundle URL, and the related UI paths invoke
those methods. `TelemetryManager::enqueue_crash_event_unconditional()`
explicitly queues a crash event despite the telemetry setting after a
crash-report/bundle send. Hiding a settings row or relying on the telemetry
toggle would therefore not satisfy a local-only promise: the relevant code,
endpoint strings and ingest path would remain in the artifact. This is static
code evidence only; none of those methods was built, called or allowed to
contact an endpoint.

The smallest acceptable project adaptation must fail closed at build time. The
local-only profile must remove or replace these external transport entry
points with local-only stubs and omit their endpoint/ingest material from the
staged artifact. The separately reviewed telemetry-contribution profile may
add back only the narrowly enumerated telemetry transport behind project-owned
consent; it may not revive crash, debug-bundle or update paths. Package checks
must inspect the selected source list/link result and staged files for the
prohibited hostnames and ingest credential, rather than testing UI visibility
alone. A root-owned `HELIX_DISABLE_AUTO_UPDATES=1` remains defense in depth for
the updater but cannot provide this compile-time boundary.

### Telemetry contribution scope to audit

The option is useful only if the user can make an informed choice. Preliminary
static inspection shows that the upstream telemetry manager has code for
session and hardware profiles, UI panel/widget usage, setting changes,
Moonraker connection stability, print outcome/context (including temperatures,
filament and file-metadata fields), errors, memory/performance samples and
update-result diagnostics. It also persists a locally generated telemetry
device identity and a queue. This is not an exhaustive approved inventory;
the project must derive and publish the final field-level inventory from the
selected source revision and capture tests before exposing consent.

The recommended first opt-in is deliberately narrow: aggregate reliability
and performance telemetry for HelixScreen development. It excludes raw logs,
G-code, camera images, printer configuration, debug bundles and automatic
crash upload. A contributor who needs to share those diagnostics should use a
separate, reviewed local-export/support flow.

### v1.0.6 telemetry field inventory and minimization decision (2026-10-27)

A static read of `TelemetryManager` at the pinned `v1.0.6` root commit shows
that its existing opt-in is materially broader than the proposed AD5M
contribution. The following is an inventory of code paths and fields, not an
assertion that an AD5M has emitted them:

| Category | Observed upstream event content |
| --- | --- |
| Correlation | persistent random UUID plus salt (sent as a deterministic hashed device ID), UTC timestamps, boot ID, schema/app version and platform |
| Host and printer profile | CPU model, architecture, OS, RAM, core count, display resolution/backend/input/backlight traits, detected model, MCU names/count, Klipper/Moonraker versions, build volume, heater/extruder/MMU/capability data, and lists of printer objects, macro, fan, stepper, LED and sensor names |
| UI and settings | theme, locale, display timing/brightness, sound/update/animation/time-format choices, home-widget IDs, panel/overlay use, performance by named panel, and old/new values for selected setting changes |
| Print context | outcome, duration, phases, filament type and millimetres, nozzle/bed temperatures, slicer, tool/AMS use, thumbnail flag and bucketed file size/duration |
| Reliability and diagnostics | process/system memory values, frame timing, Moonraker connectivity, update-result metadata, error category/code/context, crash register/memory-map fields and asynchronous-lifetime counters |

The upstream implementation creates `telemetry_device.json` with its random
identity during initialization, before it loads the user's enabled setting,
and keeps frame samples warm while disabled. It also persists queued events in
`telemetry_queue.json`. Its generic `error.context` is not a safe data
boundary: inspected callers pass caught exception text, LVGL messages, a
G-code-viewer resource path and a hexadecimal backtrace. The normal print
telemetry does not itself serialize G-code contents, images or a camera frame,
but those findings mean the project must not promise that the unmodified
upstream manager excludes all sensitive diagnostic context. Debug bundles are
separate and even broader; they remain prohibited.

Consequently, an AD5M telemetry contribution cannot expose the upstream
`telemetry_enabled` toggle or reuse its event queue/identity lifecycle. It
needs a project-owned, strictly allowlisted event schema and sender. The first
candidate schema is limited to release/platform compatibility, explicitly
coarsened UI performance/reliability counters and fixed error *codes*; it
excludes persistent identity before consent, all raw/free-form error context,
hardware/configuration names, setting old/new values, printer-object/macro
inventories, print file/filament/temperature details, update diagnostics,
memory maps and backtraces. Exact fields, retention and send cadence still
need a separate review before implementation. On decline or revocation, the
project sender must have no identifier, queue or DNS-capable transport to run.

### v1.0.6 production-candidate disposition (2026-10-27)

A fresh read-only checkout of the exact official root commit confirmed that a
small project adaptation cannot currently establish the required compile-time
local-only boundary.  `APP_SRCS` starts as a recursive wildcard over the
application source tree.  At the root Makefile level it filters mock sources
and the remote-control directory, but it has no equivalent filters or feature
gate for `update_checker.cpp`, `telemetry_manager.cpp`, `crash_reporter.cpp`,
`debug_bundle_collector.cpp`, their UI modals, or the dependent application
initialization and subject-observer call sites.

The normal application explicitly initializes UpdateChecker, CrashReporter and
TelemetryManager; its UI and diagnostics code also directly references the
same components.  Removing only the four transport implementation files would
therefore leave an unreviewed and potentially broad set of stub, UI, lifecycle
and diagnostics changes to design.  Keeping them while hiding controls or
relying on `HELIX_DISABLE_AUTO_UPDATES=1` would leave endpoint material and
egress-capable transport in the artifact.  Neither outcome meets this task.

Accordingly, the official `v1.0.6` candidate is **not approved for the first
AD5M local-only production package or service**.  This is a scope and security
boundary decision, not an allegation of malicious intent or a judgment about
upstream quality.  It does not reject HelixScreen as the project's chosen UI:
it requires a later pinned upstream revision with a demonstrably narrow
local-only profile, or an independently reviewed, maintainable upstream-
compatible adaptation under AD5M-023.  Until then, no binary, `--test` mock,
installer, package, service, or target build from this candidate is accepted.

## Repositories affected

- ad5m-project
- ad5m_enhanced_mod

## Requirements

- Treat outbound network access as a product capability, not as a side effect
  of a UI package. Local-only is the release default.
- Keep the local Moonraker WebSocket/HTTP connection functional. A prohibition
  on third-party egress must not accidentally block same-device printer control.
- Build with `HELIX_PACKAGING=1`, `ENABLE_REMOTE_CONTROL=no` and
  `ENABLE_MOCKS=no` explicitly; prove the remote-control server/client source
  is absent from the staged binary.
- Telemetry may be offered solely as a separately named, explicit development
  contribution option. Before enabling it, the touchscreen must show the
  receiving organization/endpoint, a concise source-derived data inventory,
  the purpose (upstream HelixScreen development), and the fact that already
  sent reports cannot be recalled. Its initial state is off; upgrades,
  configuration imports, a Moonraker request, and environment inheritance may
  not silently change it to on.
- Telemetry consent and revocation must be device-local and intentional. A
  user must be able to turn it off from the touchscreen without network access;
  that immediately stops DNS/HTTP activity and removes queued unsent telemetry
  and its local telemetry device identifier unless the user expressly chooses
  to retain them. The UI must distinguish "off", "queued locally" and
  "sent"; it must not claim deletion of events already received upstream.
- Do not rely solely on the upstream mutable `telemetry_enabled` setting.
  A reviewed project gate must make its transport unreachable until the AD5M
  opt-in state is present. The telemetry-enabled profile may retain the
  telemetry endpoint and its write-only ingest credential; the local-only
  profile must not. Neither profile may retain unrelated external transport.
- Do not reuse the pinned upstream telemetry manager's identity, queue or
  event schema for the AD5M contribution. It creates local identity before
  consent and can emit broad profiles and free-form diagnostic context. Use a
  project-owned, strict allowlist with no persistent identifier or queued
  report before consent; consent/revocation must create/remove the complete
  local contribution state atomically.
- Disable the upstream update checker and self-updater as a project-owned,
  upgrade-safe policy. `HELIX_DISABLE_AUTO_UPDATES=1` in a root-owned service
  environment may be a short-term defense-in-depth measure, but it is not by
  itself sufficient evidence that future upstream code cannot egress. The
  package needs a reviewed compile/staging adaptation or a testable upstream
  build switch that removes the automatic check and update UI. Any future
  project-managed update path is AD5M-019; it must not inherit upstream
  self-update authority.
- Do not ship upstream crash-report or debug-bundle upload actions in the
  local-only variant. Preserve local crash evidence and offer an export path
  only after reviewing data redaction, ownership and user interaction; never
  auto-upload it.
- Treat the upstream ingest credential as a capability, not a secret. It may
  occur only in the reviewed telemetry-enabled profile and must remain absent
  from local-only artifacts. Its presence never authorizes crash uploads,
  debug bundles or self-updates.
- A future opt-in external-support, crash upload or update feature requires
  its own ADR, privacy/data inventory, explicit consent and revocation design,
  endpoint ownership review, failure behavior, and software-only evidence.
  It must not silently re-enable an upstream service.

## Non-goals

- Blocking local Moonraker access, user-selected LAN printer access, or a
  separately designed project update mechanism.
- Operating an AD5M, contacting an upstream telemetry/crash service, or
  attempting an upstream self-update.
- Promising anonymous diagnostics without a data inventory and review.
- Treating telemetry consent as consent to crash reports, debug bundles,
  third-party support access or an updater.

## Proposed implementation

1. Make a minimal, auditable Buildroot-only HelixScreen adaptation with an
   explicit local-only feature gate and a narrowly scoped telemetry opt-in
   gate. It must be pinned with the HelixScreen source and apply cleanly only
   to the reviewed commit.
2. Set a root-owned, fixed service environment as a second layer: suppress
   upstream updates, pin the local Moonraker URL, and do not inherit arbitrary
   environment variables from an interactive shell.
3. Remove or compile out upstream external update, crash-upload and
   debug-bundle-upload entry points. Retain only local crash files and local
   export after its separate review. Gate the upstream telemetry transport
   behind project-owned consent state; do not use its setting page unchanged.
4. Add profile-aware package checks. The local-only artifact must reject all
   known external endpoint hostnames and the ingest credential. The
   telemetry-enabled artifact may contain only the reviewed telemetry endpoint
   and credential, while still rejecting update and crash/debug-bundle hosts.
   Check the generated service environment and selected source list too.
5. Exercise the staged binary against a local mock Moonraker endpoint while
   outbound connections are denied. Record local connection, idle operation,
   a disabled-update state and a disabled telemetry state separately from any
   physical-printer test. Exercise the telemetry opt-in only against a local
   test endpoint or other non-production fixture; this project must not use a
   real user/device or public endpoint as a test fixture.

## Acceptance criteria

- The staged local-only application neither contains nor can invoke the
  upstream remote-control server/client, updater, telemetry transport, crash
  uploader or debug-bundle uploader.
- If a telemetry-enabled profile is accepted, it can reach only its reviewed
  telemetry endpoint after current-device touchscreen consent. It cannot reach
  GitHub/R2 update, crash-report or debug-bundle endpoints and it does not
  enable remote control.
- Static package checks reject `crash.helixscreen.org`, GitHub release
  endpoints and the upstream ingest credential in local-only runtime files.
  The telemetry-enabled profile may make a narrowly enumerated exception for
  `telemetry.helixscreen.org` and its associated write-only credential.
- The generated service environment is root-owned, explicit and does not rely
  on user-editable defaults to keep external egress off.
- A software-only test with outbound access denied proves that the UI can
  connect to a local mock Moonraker service and stay operational without an
  update/telemetry request.
- Tests prove that declining telemetry makes no DNS or HTTP request, that
  revocation stops it and clears the stated local state, and that the opt-in
  transport is constrained to a non-production fixture.
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

- AD5M-023 owns the decision and, if justified, implementation of an
  upstream-compatible compile-time local-only profile.  The pinned candidate
  has no sufficiently narrow switch beyond remote control.
- Produce the upstream telemetry data inventory and consent/revocation copy
  before exposing its toggle. Decide whether the opt-in is a separate package
  profile or one explicitly gated runtime capability.
- Inventory all staged executable and non-executable runtime files before
  choosing the endpoint/credential scan.
- Define a no-egress test harness that still permits an explicit local
  Moonraker fixture.
