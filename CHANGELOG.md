# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.0] - 2026-07-30

### Added

- `--add-hosts`: after a host's service scan, extracts domain names from LDAP/TLS/HTTP-redirect
  output and offers to append them to `/etc/hosts` (asks first, `sudo tee`).
- `--misconfig`: checks open ports for common misconfigurations — anon FTP, null-session SMB/RPC
  enum, NFS exports, default/empty creds on MySQL/MSSQL/Redis/Mongo/CouchDB/Cassandra, open
  VNC/X11/Modbus/Docker, HTTP default accounts. Rate-capped; findings surface in `SUMMARY.md`.

## [1.1.0] - 2026-07-30

### Added

- Post-sweep rate-limit detection: cross-checks the pre-sweep probe's confirmed-open ports against
  the sweep results, rechecks any that went missing, and folds recovered ports back into the
  service scan.

### Fixed

- `ctf` mode no longer throttles scan rate when the defense probe detects inspection — that
  defeated its purpose. It keeps the extra cooldown and the rate-limit check above instead.
- Boost multiplier reduced from 3x to 2x.

## [1.0.0] - 2026-07-30

Initial public release.

### Modes

`ctf`, `normal`, `fast`, `stealth` — layered discovery, SYN sweep, service/version scan, OS
fingerprint, per-mode timing and rate profiles.

### Adaptive tuning

Pre-sweep badsum differential probe classifies the target as defended or undefended and scales
rate, timing, retries, and inter-phase cooldown automatically. Automatic retry when a service scan
reports all-filtered results despite the sweep already proving those ports open.

### CIDR handling

Targets broader than `/16` auto-chunk into `/24` blocks, scanned as resumable child runs with a
live ETA computed from measured chunk duration. `/8`-or-broader requires `--allow-huge-scope`.
Targets broader than `/16` require `-e`/`--interface`.

### Resumability

Every phase checkpoints to disk; every flag from the original invocation persists, so `-r <outdir>`
alone replays the exact same scan. Clean `SIGINT`/`SIGTERM` handling.

### Scan control

`--ports`, `--mac`, `--masscan`, `--vuln`, `--udp`, `--badsum-test`, `--cve`, `--ot`,
`--sec-headers`.

### Evasion

`--idle-zombie`, `--firewall-map`, `--paranoid`, `--ttl`, `--proxy`.

### Configuration

YAML config auto-loaded from `./nigoh.yaml` or via `--config`; CLI flags always win. Example
profiles in `examples/`.

### Reporting

`--diff`, `--webhook`, consolidated `SUMMARY.md`, plain-text run journal.

[Unreleased]: https://github.com/oxbyte0/nigoh/compare/v1.2.0...HEAD
[1.2.0]: https://github.com/oxbyte0/nigoh/compare/v1.1.0...v1.2.0
[1.1.0]: https://github.com/oxbyte0/nigoh/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/oxbyte0/nigoh/releases/tag/v1.0.0
