# Security Policy

## Scope

This policy covers vulnerabilities in `nigoh` itself — the wrapper script — such as:

- Command injection via unsanitized target, port, or file-path input
- Unsafe handling of temporary files or the state/journal files it writes
- Privilege-escalation issues related to its use of `sudo` (`--masscan`, `--add-hosts`)
- Logic errors that cause the tool to scan or exclude the wrong hosts

It does **not** cover vulnerabilities in `nmap`, `masscan`, `yq`, or any NSE script `nigoh` invokes.
Report those to the respective upstream projects.

## Known Risk: Auto-Loaded Config File

`nigoh` automatically loads `./nigoh.yaml` from the current directory if present, with no prompt.
Config values only ever populate the same fixed set of scan parameters a CLI flag would (target,
mode, opt-in flags) — the file's content is never sourced or evaluated as shell, so this is not a
code-execution vector. It is, however, a data-integrity one: running `nigoh` inside a directory an
attacker controls could silently redirect the scan target, or point `--webhook` at an attacker's
URL to exfiltrate results. Review a directory's contents, including hidden config files, before
running `nigoh` there without an explicit `-t`/`--target` overriding whatever the config supplies.

## Known Risk: `--update` Replaces the Running Script from GitHub over HTTPS

`--update` (and the background check with `auto_update: true`) fetches the current `nigoh` script
from `main` on GitHub over HTTPS and, after a `bash -n` syntax check, overwrites the file the
`nigoh` command resolves to. There is no signature or independent checksum verification — the
trust boundary is the HTTPS connection to GitHub and control of the `oxbyte0/nigoh` repository,
the same trust model as `git pull` or `curl | bash` from this project. The syntax check catches
corrupted or truncated downloads, not a maliciously-modified but syntactically valid script. Do not
enable `auto_update: true` in an environment where an unattended silent script replacement from a
third-party-controlled network path would be unacceptable.

## Known Risk: `--misconfig` Performs Live Auth Attempts

Several scripts run by `--misconfig` (`mysql-empty-password`, `ms-sql-empty-password`,
`http-default-accounts`, `smb-enum-shares`/`smb-enum-users`) are nmap NSE-`intrusive`, not `safe` —
they make a single live authentication attempt (empty/default credentials) or null-session
enumeration against the target rather than just reading a banner. This is standard misconfiguration
discovery, not brute forcing, but it is active interaction with the target's auth layer and should
only be run against systems you're authorized to test in that manner.

## Supported Versions

| Version | Supported |
|---|---|
| Latest release on `main` | Yes |
| Older tagged releases | No |

## Reporting a Vulnerability

Please do not open a public issue for security vulnerabilities.

If this repository has GitHub's private vulnerability reporting enabled (Settings → Security →
"Report a vulnerability"), use that channel. Otherwise, contact the maintainer directly through
the contact information listed on their GitHub profile.

Please include:

- A description of the issue and its potential impact
- Steps to reproduce, including the exact `nigoh` invocation and environment
- Any relevant script output (with `--debug` if applicable)

Reports are acknowledged as soon as practical, and a fix or mitigation is prioritized based on
severity. Credit is given in the changelog for responsibly disclosed issues, unless the reporter
requests otherwise.

## Responsible Use Disclosure

`nigoh` is an offensive-security reconnaissance tool. It is provided for use exclusively against
systems the operator owns or is explicitly authorized to test. This is a condition of use, not a
vulnerability to be reported — see [README.md § Responsible Use](README.md#responsible-use).
