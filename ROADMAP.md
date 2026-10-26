# AD5M Project Roadmap

This file is a high-level backlog.

Actual implementation work should use scoped task files under tasks/.

## Reliability

- pause and filament-runout timeout behavior
- MCU timeout investigation
- G-code transfer/upload integrity
- persistent user configuration
- safer installer/update mechanism
- rollback/recovery
- MCU update recovery

## Safety

- review heater verification configuration
- review temperature limits
- probing/load-cell safety
- MCU flashing failure handling
- shutdown/power behavior

## Bed leveling and nozzle preparation

- native adaptive bed mesh
- remove unnecessary thermal cycling during mesh preparation
- improve nozzle cleaning before probing
- optional physical nozzle brush/wiper
- retain explicit full-bed calibration fallback

## UI

- HelixScreen integration
- Wi-Fi/network configuration
- filament load workflow
- filament unload workflow
- automatic preheat for extrude/retract
- purge speed units/defaults
- stale filament requests/prompts
- diagnostics/status display

## Buildroot and system

- official Buildroot 2025.02 LTS maintenance
- audit historical AD5M Buildroot modifications
- dependency completeness
- memory/resource management
- logging limits
- possible zram/swap experiments
- service startup optimization

## Security

- Moonraker authorization
- trusted_clients policy
- CORS policy
- SSH credential handling
- firewall/network exposure
- release integrity
- SHA-256/signing

## Tooling

- CI
- automated configuration validation
- shell/Python linting
- reproducible build documentation
- component/version manifest
- diagnostics export
