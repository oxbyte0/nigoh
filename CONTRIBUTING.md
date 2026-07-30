# Contributing to nigoh

Contributions are welcome, from bug fixes to new opt-in scan profiles. This document describes the
expectations for a change to be merged.

## Before You Start

For anything beyond a small fix, open an issue first to discuss the approach. This avoids wasted
effort on a change that doesn't fit the project's scope — `nigoh` is deliberately a single-file,
dependency-light `nmap` wrapper, not a general-purpose scanning framework.

## Development Setup

`nigoh` has no build step. Clone the repository and run the script directly:

```sh
git clone https://github.com/oxbyte0/nigoh.git
cd nigoh
chmod +x nigoh
./nigoh -h
```

## Coding Standards

- **Bash only**, targeting Bash 4.3+. No external language runtimes.
- **`shellcheck -x nigoh` must report zero warnings** before a change is submitted. This is
  non-negotiable — several past regressions in this project were `set -e` interactions that
  shellcheck's `--severity=style` catches.
- Avoid `[[ condition ]] && action` as the last statement in a loop body. Under `set -e`, a false
  condition on the final iteration makes the loop's own exit status non-zero, which silently
  terminates the entire script. Use `if [[ condition ]]; then action; fi` instead.
- Guard every glob that might not match (`for f in dir/*.ext`) with an existence check
  (`[[ -e "$f" ]] || continue`), and guard command substitutions that might legitimately fail
  (`var=$(cmd) || true`) rather than letting `set -e` propagate an expected empty result as a
  fatal error.
- New comments should explain **why**, not **what**. If a comment restates what the code already
  makes obvious, remove it.
- New phases should follow the existing pattern: a `CURRENT_PHASE` assignment, a `PHASE_*_DONE`
  guard if the phase should not repeat on resume, a call to `save_state` and `journal_append`
  after completion, and a corresponding entry in `save_state()`'s persisted variable list if the
  phase is gated by a new CLI flag.
- A new CLI flag touches **three** places, not one: the `save_state()` persistence list, and
  `run_chunk_child()`'s forwarded-args list (so a broad-CIDR chunked scan honors it too). Missing
  the second one is easy and silent — it only shows up as a chunk child behaving like the flag was
  never passed. Cross-check `save_state()`'s variable list against `run_chunk_child()`'s whenever
  either changes.

## Testing a Change

There is no formal test suite; verification is done by dry-run execution.

```sh
NIGOH_DRYRUN=1 NIGOH_NO_BANNER=1 nigoh -t 10.0.0.1 -m ctf -o /tmp/test-run
```

`NIGOH_DRYRUN=1` replaces every `nmap` invocation with a logged no-op, so the full phase pipeline,
argument construction, and state machine can be exercised without a real target or elevated
privileges. Useful additional variables:

| Variable | Effect |
|---|---|
| `NIGOH_DRYRUN=1` | Simulate `nmap` calls instead of executing them |
| `NIGOH_DRYRUN_SLEEP=<seconds>` | Duration of each simulated `nmap` call (default: 0.2) |
| `NIGOH_DRYRUN_FAIL_COUNT=<n>` | Simulate the first `n` attempts of a call failing, to exercise `run_nmap`'s retry logic |
| `NIGOH_NO_BANNER=1` | Suppress the startup banner, useful for scripted test output (`-q`/`--quiet` does the same as a real flag) |

For a change touching argument parsing, state persistence, or the phase pipeline, verify at
minimum:

1. `bash -n nigoh` and `shellcheck -x nigoh` are both clean. Also run `shellcheck --severity=style`
   — several real bugs in this project were only caught at that severity.
2. A fresh run with the new/changed flag completes with exit code 0 under `NIGOH_DRYRUN=1`. Check
   `$?` directly; do not pipe through `grep`/`head` to eyeball success, since a silent crash and a
   truncated-but-successful run look identical under a filtered pipe. This has produced false
   passes in this project's own history more than once.
3. An interrupted run (`SIGINT` mid-scan, from a shell with `set -m` — job control matters, see
   below) saves state and resumes correctly with `-r`, and the resumed run still honors the
   original flag — every opt-in flag is persisted in `save_state()` specifically so this holds.
4. If the target/CIDR sizing logic is touched, verify a chunked scan (any target broader than
   `/16`) still forwards the flag to its chunk children — see the coding-standards note above.
5. The change does not alter behavior for modes/flags it wasn't intended to touch — run the full
   flag combination sweep described in the pull request template.

Note on `SIGINT` testing: a backgrounded job (`cmd &`) in a shell without job control (`set -m`)
has `SIGINT`/`SIGQUIT` pre-ignored by bash itself before the script's own `trap` ever runs, which
is a shell-level POSIX behavior, not a bug in `nigoh`. Always test interrupts with `set -m` active
in the test shell, or you will see the trap appear to silently not fire.

## Submitting a Pull Request

1. Fork the repository and create a feature branch.
2. Keep the diff focused on a single logical change.
3. Update `README.md` and the in-script `--help` text if the change adds, removes, or alters a
   user-facing flag.
4. Add an entry to `CHANGELOG.md` under `[Unreleased]`.
5. Open the pull request with a description of what changed and how it was verified.
