# nigoh

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Shell: Bash](https://img.shields.io/badge/Shell-Bash-4EAA25?logo=gnubash&logoColor=white)](https://www.gnu.org/software/bash/)
[![Requires: nmap](https://img.shields.io/badge/Requires-nmap-informational)](https://nmap.org)
[![ShellCheck](https://img.shields.io/badge/ShellCheck-passing-brightgreen)](https://www.shellcheck.net/)

**A mode-driven, self-tuning, resumable nmap automation wrapper.**

*Нигоҳ (nigoh) — Tajik-Persian for "gaze" or "watch." The project's own motto: nothing unseen.*

`nigoh` turns a full reconnaissance pipeline — host discovery, port sweep, service and version
detection, OS fingerprinting, and reporting — into a single command. It checkpoints after every
phase, resumes exactly where it left off, and adjusts its own scan aggression based on whether the
target appears to be actively defended. It is a single, dependency-light Bash script built on top
of `nmap`.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Scan Modes](#scan-modes)
- [Command-Line Reference](#command-line-reference)
- [Adaptive Defense Tuning](#adaptive-defense-tuning)
- [CIDR Sizing and Chunked Scanning](#cidr-sizing-and-chunked-scanning)
- [Configuration File](#configuration-file)
- [Resumability and State Persistence](#resumability-and-state-persistence)
- [Output Structure](#output-structure)
- [Pipeline Phases](#pipeline-phases)
- [Usage Examples](#usage-examples)
- [Responsible Use](#responsible-use)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)
- [Further Reading](#further-reading)
- [Author](#author)

---

## Overview

Manual nmap usage during an assessment tends to accumulate the same friction repeatedly: retyping
timing and rate flags for every host, forgetting `-Pn` after a failed ping sweep, or misreading a
target's rate-limiting response as a closed port. `nigoh` addresses this by encoding a set of
well-defined operating modes, an adaptive pre-sweep probe, and a durable checkpoint/resume system
into a single tool, so that scan behavior is consistent, repeatable, and safe to interrupt.

`nigoh` is not a scanner in its own right — it is an orchestration layer over `nmap`. All actual
packet-level work is performed by `nmap`; `nigoh` is responsible for sequencing, tuning, retrying,
and reporting.

## Key Features

| Capability | Description |
|---|---|
| **Mode-driven scanning** | Four operating modes (`ctf`, `normal`, `fast`, `stealth`) encode distinct aggression and timing profiles suited to different engagement contexts. |
| **Adaptive defense tuning** | A lightweight pre-sweep probe classifies the target as defended or undefended and scales rate, timing, retries, and cooldown accordingly. |
| **Full resumability** | Every phase checkpoints to disk. A run interrupted by `Ctrl+C`, a dropped connection, or a crash resumes from the exact point it stopped, with every original flag intact. |
| **Automatic false-filtered detection** | If a service scan reports ports as filtered immediately after a sweep proved them open, `nigoh` treats this as a rate-limit artifact, backs off, and retries automatically. |
| **Firewall policy mapping** | An optional ACK/Window/Maimon differential pass characterizes firewall behavior before committing to a full sweep. |
| **ICS/OT-aware discovery** | An optional ultra-conservative probe profile for Modbus, S7, and BACnet, suited to industrial control environments. |
| **CVE attribution** | Optional best-effort CVE matching against detected service versions via the `vulners`/`vulscan` NSE scripts. |
| **Change detection** | Optional `ndiff`-based comparison against a prior run to surface what changed between assessments. |
| **Result delivery** | Optional webhook delivery of the run summary to a SIEM, ticketing system, or chat integration. |
| **Native audit trail** | Every run directory contains a plain-text, append-only phase journal, independent of any version control system. |

## Requirements

| Component | Requirement |
|---|---|
| Shell | Bash 4.3+ (uses `local -n` namerefs) |
| Core dependency | [nmap](https://nmap.org) |
| Optional | [masscan](https://github.com/robertdavidgraham/masscan) — `--masscan` |
| Optional | `curl` — `--webhook` |
| Optional | `ndiff` (bundled with nmap) — `--diff` |
| Optional | `vulners` or `vulscan` NSE scripts — `--cve` |
| Optional | [`yq`](https://github.com/kislyuk/yq) — `--config` / auto-loaded `nigoh.yaml` |
| Optional | `ip` (iproute2, present on virtually every Linux install) — interface listing for `-e` |

## Installation

```sh
git clone https://github.com/oxbyte0/nigoh.git
cd nigoh
chmod +x nigoh
sudo ln -s "$(pwd)/nigoh" /usr/local/bin/nigoh   # optional: place on PATH
```

Verify the installation:

```sh
nigoh -h
```

## Quick Start

```sh
nigoh -t 10.10.11.5 -m ctf              # single host, minimal overhead
nigoh -t 10.10.20.0/24 -m normal        # subnet, layered discovery
nigoh -r nigoh_ctf_20260730_095910      # resume a previous run
```

Every run opens with a startup banner, randomly picked from `banners/` if present (falling back to
a built-in one otherwise). Pass `-q`/`--quiet`, or set `NIGOH_NO_BANNER=1`, to suppress it — useful
in scripts and CI.

## Scan Modes

| Mode | Intended use case | Behavior |
|---|---|---|
| `ctf` | A single known-up host (CTF, HTB, a lab box) | Skips host discovery, runs a full `-p-` SYN sweep at `--min-rate 5000 -T4`, followed by version and OS detection |
| `normal` | Default. A real engagement or an unknown network shape | Ping sweep, confirmation, then a moderated SYN sweep and service scan (`-T3`, `--defeat-rst-ratelimit`) |
| `fast` | Time-constrained assessment where noise is an acceptable trade-off | Maximum sweep rate (`--min-rate 10000`), highest version-detection intensity, two-pass OS detection |
| `stealth` | An actively monitored environment | Fragmented packets, decoy hosts, spoofed source port, `-T2`, host randomization, inter-probe delay |

`ctf` mode rejects CIDR or range targets by design: it has no discovery phase, and would otherwise
issue a full-rate blind sweep against every address in the range.

## Command-Line Reference

### Target & Output

| Flag | Argument | Description |
|---|---|---|
| `-t`, `--target` | `<host\|CIDR>` | Scan target. Single host for `ctf`; host or CIDR/range for the other modes. |
| `-m`, `--mode` | `<mode>` | `ctf`\|`normal`\|`fast`\|`stealth` (default: `normal`) |
| `-o`, `--out` | `<dir>` | Output directory (default: `nigoh_<mode>_<timestamp>`) |
| `-r`, `--resume` | `<dir>` | Resume a previous run, restoring every original flag |
| `-x`, `--exclude` | `<file>` | Hosts/CIDRs to exclude from scanning |
| `-e`, `--interface` | `<name>` | Egress interface for all `nmap` traffic. Required once the target is broader than `/16` — see [CIDR Sizing](#cidr-sizing-and-chunked-scanning) |

### Scan Control

| Flag | Argument | Description |
|---|---|---|
| `-p`, `--ports` | `<list>` | Scan these ports directly, bypassing discovery and the sweep |
| `--mac` | `<vendor>` | Spoof the source MAC vendor prefix (stealth/LAN only) |
| `--masscan` | — | Seed the sweep with a masscan pass before nmap confirms |
| `--vuln` | — | Run nmap's safe vulnerability-category NSE scripts, rate-capped |
| `--udp` | — | Scan the top 200 UDP ports |
| `--badsum-test` | — | One-off diagnostic distinguishing stateful from stateless filtering |
| `--cve` | — | Match detected service versions against known CVEs (requires `vulners`/`vulscan`) |
| `--ot` | — | ICS/OT-safe discovery profile: Modbus, S7, BACnet |
| `--sec-headers` | — | Checks discovered HTTP(S) services for security-relevant response headers (HSTS, CSP, X-Frame-Options, etc.) via nmap's bundled `http-security-headers` script |

### Evasion

| Flag | Argument | Description |
|---|---|---|
| `--idle-zombie` | `<host>` | Blind idle scan (`-sI`); the specified host communicates with the target, not the operator |
| `--firewall-map` | — | ACK/Window/Maimon differential pass to characterize firewall policy before the sweep |
| `--paranoid` | — | Forces `-T0` and a 60-second minimum cooldown, overriding mode defaults and adaptive tuning |
| `--ttl` | `<n>` | Spoofs the IP TTL (1-255) — blend with an expected OS baseline or dodge TTL-anomaly detection |
| `--proxy` | `<url>` | Chains through a SOCKS4 or HTTP proxy (nmap's native `--proxies`). Forces a TCP connect scan (`-sT`), since proxying only works over real TCP connections, not raw SYN |

Stealth mode's packet padding (`--data-length`) is randomized once per run (20-219 bytes) rather
than fixed, so it doesn't itself become a recognizable, constant-size signature across runs.

### Advanced

| Flag | Argument | Description |
|---|---|---|
| `--recurse-pivot` | — | Automatically scans subnets discovered via traceroute (depth-capped at one hop) |
| `--keep-runs` | `<n>` | Number of prior run directories to retain (default: 20) |
| `--diff` | `<dir>` | Compares this run's sweep against a prior run directory |
| `--webhook` | `<url>` | Delivers the run summary via HTTP POST on completion |
| `--allow-huge-scope` | — | Permits a `/8`-or-broader target (still auto-chunked into `/24` blocks) — see [CIDR Sizing](#cidr-sizing-and-chunked-scanning) |
| `--config` | `<file>` | Loads defaults from a YAML file. `./nigoh.yaml` is auto-loaded if present; CLI flags always take precedence — see [Configuration File](#configuration-file) |
| `--debug` | — | Enables verbose internal logging |

### Utility

| Flag | Description |
|---|---|
| `-q`, `--quiet` | Suppresses the startup banner |
| `--list` | Displays run history from `~/.nigoh/history.log` |
| `-h`, `--help` | Displays command-line help |

## Adaptive Defense Tuning

Before the primary sweep, `nigoh` performs a **badsum differential probe**: a small number of SYN
packets carrying a deliberately invalid TCP checksum, sent to a handful of common ports.

- A packet filter that inspects only IP/TCP header fields without validating the checksum will
  typically still respond.
- A stateful firewall or intrusion detection system performing deeper inspection will typically
  drop the malformed packet silently.

Based on the ratio of dropped to answered probes, `nigoh` classifies the target and adjusts scan
parameters automatically:

| Classification | Sweep and service-scan adjustment |
|---|---|
| Active inspection detected | Rate divided by 4, timing capped at `-T3`, retries and version intensity halved, scan delay added, inter-phase cooldown doubled |
| No active inspection detected | Rate multiplied by 3 (capped), version-detection intensity maximized, inter-phase cooldown halved |

This mechanism exists to prevent a specific and previously observed failure mode: an aggressive
full-port sweep triggering a target's rate limiter, followed immediately by a service scan that
misreads the resulting block window as closed ports rather than a transient filtering artifact. A
mandatory cooldown between the sweep and the service scan, combined with automatic retry logic when
a service scan unexpectedly reports all-filtered results, closes this gap independently of the
adaptive probe.

The probe is skipped in `stealth` mode, which already assumes an actively monitored target and is
tuned accordingly from the outset.

## CIDR Sizing and Chunked Scanning

`nigoh` sizes any target given in literal `a.b.c.d/N` notation and applies one of three tiers.
Comma-separated lists and octet ranges (`10.0.0.1-50`) have no single prefix to size and are
unaffected — they pass through to `nmap` exactly as before.

| Prefix | Behavior |
|---|---|
| `/16` or narrower (more specific) | Unchanged: a single `nmap` invocation, as always |
| `/9` – `/15` | Automatically split into `/24` blocks and scanned as a sequence of independent, resumable child runs |
| `/8` or broader | Refused by default; requires `--allow-huge-scope`, and is still chunked into `/24` blocks — never attempted as one flat sweep |

A target broader than `/16` — in either tier — additionally **requires `-e`/`--interface`**. This
is a deliberate safety gate: a target that size is exactly the shape of a typo'd octet or an
accidentally-inherited default route, and naming the interface explicitly turns "which network am
I about to scan" into a stated decision rather than whatever the kernel's routing table happens to
pick. If the interface exists but the flag is misspelled or the interface has no IPv4 address,
`nigoh` warns (listing available interfaces) but does not block the scan, since a routed egress
(e.g. through a VPN tunnel to a genuinely remote network) is a legitimate configuration.

Chunked runs behave like any other `nigoh` run with respect to interruption and resume — the
parent orchestrator tracks which `/24` blocks have completed, and `-r <outdir>` picks up from the
first incomplete one, skipping everything already done. Each block also reports a live progress
line with an **ETA computed from the measured average duration of the blocks already completed in
this run** — never a pre-scan guess:

```
[+] chunk 3/512 (0%): 10.60.2.0/24 — ETA: 8m31s remaining (avg 1s/chunk)
```

Network-boundary math is exact regardless of how the target address is written — a host address
within a range (`192.168.15.20/19`) or a non-aligned base (`172.20.99.200/12`) both resolve to the
correct underlying network before any scanning or chunking happens.

## Configuration File

Frequently-used flags can be captured in a YAML file instead of retyped on every invocation.
`./nigoh.yaml` in the current directory is loaded automatically if present; an explicit path can
be given with `--config <file>`. **Command-line flags always take precedence** — a config file
supplies defaults, not overrides.

```yaml
target: 10.44.0.0/16
mode: normal
interface: eth1
udp: true
cve: true
webhook: https://hooks.example.com/incoming/nigoh
```

See [`nigoh.example.yaml`](nigoh.example.yaml) in this repository for the full set of supported
keys with inline documentation. The [`examples/`](examples/) directory has ready-to-copy, scenario-specific
profiles:

| File | Scenario |
|---|---|
| [`examples/ctf.yaml`](examples/ctf.yaml) | A single CTF/HTB box — fast, CVE matching on, minimal else |
| [`examples/cidr-simple.yaml`](examples/cidr-simple.yaml) | An ordinary internal subnet (`/16` or narrower), nothing unusual |
| [`examples/cidr-hardened.yaml`](examples/cidr-hardened.yaml) | A large corporate range behind a real IDS/IPS — interface, chunking, firewall mapping, and paranoid timing all set deliberately |
| [`examples/universal.yaml`](examples/universal.yaml) | An unfamiliar environment — the balanced default to start from when you don't yet know what you're dealing with |

Parsing requires [`yq`](https://github.com/kislyuk/yq). If `./nigoh.yaml` is found but `yq` isn't
installed, `nigoh` warns and proceeds without it; an explicit `--config` with `yq` missing is a
hard error, since that was a deliberate request rather than an opportunistic default. Only a fixed,
known set of keys is ever read — unknown keys are ignored, and the file's content is never
evaluated as shell.

## Resumability and State Persistence

Every phase writes a checkpoint to `.nigoh_state` in the run directory immediately upon completion.
This checkpoint includes **every flag from the original invocation** — target, mode, port
overrides, exclusions, interface, and all opt-in flags (`--udp`, `--vuln`, `--cve`, `--paranoid`,
`--firewall-map`, `--ot`, `--idle-zombie`, `--diff`, `--webhook`, `--mac`, `--keep-runs`,
`--masscan`, `--recurse-pivot`, `--allow-huge-scope`, `--ttl`, `--proxy`, `--sec-headers`) — not
merely the target and mode.

Consequently:

```sh
nigoh -t 10.10.11.5 -m ctf --paranoid --cve --udp
# interrupted (Ctrl+C, dropped VPN, terminated session, etc.)
nigoh -r nigoh_ctf_20260730_095910
```

resumes the identical scan configuration — paranoid timing, CVE matching, and the UDP sweep all
remain active — rather than reverting to an unqualified default run.

A `SIGINT`/`SIGTERM` handler terminates any in-flight `nmap` child process cleanly and persists
state before exiting, so an interrupted scan is always in a consistent, resumable state.

## Output Structure

```
nigoh_<mode>_<timestamp>/
├── discovery/       ping sweep, defense probe, firewall map, OT probe
├── sweep/           full TCP SYN sweep output, masscan seed data if used
├── service/         per-host service/version/script scan output
├── evasion/         secondary evasion pass (stealth mode)
├── vuln/            --vuln NSE output
├── udp/             --udp top-200 sweep output
├── os/              OS fingerprint and traceroute output
├── live_hosts.txt   resolved target list
├── SUMMARY.md        consolidated results
├── .nigoh_state     resume checkpoint (not intended for manual editing)
└── .nigoh_journal   plain-text, append-only phase timeline
```

## Pipeline Phases

| # | Phase | Trigger |
|---|---|---|
| 1 | Host discovery | Automatic (CIDR targets only; skipped in `ctf` mode) |
| 1b | Interactive host/mode selection | CIDR target with no `-m` specified |
| 1c | Adaptive defense probe | Automatic (all modes except `stealth`) |
| 1d | Firewall policy mapping | `--firewall-map` |
| 1e | ICS/OT discovery | `--ot` |
| 2 | Full TCP SYN sweep | Automatic (skipped when `-p` is supplied) |
| 2b | Change detection | `--diff` |
| 3 | Service/version/script scan | Automatic |
| 4 | Evasion pass | Automatic in `stealth` mode |
| 5 | Vulnerability scan | `--vuln` |
| 6 | UDP sweep | `--udp` |
| 7 | OS fingerprint and traceroute | Automatic |
| 7b | Recursive pivot expansion | `--recurse-pivot` |
| 8 | Badsum diagnostic | `--badsum-test` |
| 9 | Consolidated report | Automatic |
| — | Webhook delivery | `--webhook` |

## Usage Examples

```sh
# Single CTF/HTB host
nigoh -t 10.10.11.5 -m ctf

# The target is rate-limiting scans; reduce aggression accordingly
nigoh -t 10.10.11.5 -m ctf --paranoid

# Subnet-wide assessment on a live engagement
nigoh -t 10.10.20.0/24 -m normal

# Maximum-depth scan with CVE attribution and UDP coverage
nigoh -t 10.10.11.5 -m fast --cve --udp

# Actively monitored environment; minimize footprint from the outset
nigoh -t 10.10.20.5 -m stealth --mac 00:0c:29

# Characterize firewall behavior before committing to a full sweep
nigoh -t 10.10.11.5 -m normal --firewall-map

# Industrial control network; use the conservative OT-aware profile
nigoh -t 10.10.30.5 -m normal --ot

# Blind scan via an intermediary host
nigoh -t 10.10.11.5 -m ctf --idle-zombie 10.10.11.99

# Ports already known; skip discovery entirely
nigoh -t 10.10.11.5 -p 22,80,443

# Re-assessment; report only what changed since the prior run
nigoh -t 10.10.11.5 -m normal --diff nigoh_normal_20260701_120000

# Deliver the summary to an external system on completion
nigoh -t 10.10.11.5 -m normal --webhook https://hooks.example.com/incoming

# Internal corp network, broader than /16 — interface required, auto-chunked into /24s
nigoh -t 10.44.0.0/14 -m normal -e eth1

# Load target/mode/flags from ./nigoh.yaml instead of the command line
nigoh
```

## Responsible Use

`nigoh` executes real network scans against real targets. It is intended exclusively for use
against systems the operator owns or is explicitly authorized to test — CTF and training
environments, personal lab infrastructure, or engagements covered by a signed scope of work.
Scanning systems without authorization is illegal in most jurisdictions. The authors accept no
responsibility for misuse of this tool.

## Contributing

Contributions are welcome. Please review [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a
pull request. All changes must pass `bash -n` and `shellcheck -x` with no warnings.

## Security

To report a security issue with `nigoh` itself, see [SECURITY.md](SECURITY.md).

## License

Released under the [MIT License](LICENSE).

## Further Reading

[CHANGELOG.md](CHANGELOG.md) tracks released versions and what changed in each.

## Author

oxbyte ([linkedin.com/in/skomron](https://linkedin.com/in/skomron) ·
[oxbyte.blog](https://oxbyte.blog))
