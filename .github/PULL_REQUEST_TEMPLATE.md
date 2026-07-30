## Summary

What does this change do, and why?

## Checklist

- [ ] `bash -n nigoh` passes
- [ ] `shellcheck -x nigoh` and `shellcheck --severity=style -x nigoh` both report zero warnings
- [ ] `README.md` and in-script `--help` updated if a user-facing flag changed
- [ ] `CHANGELOG.md` updated under `[Unreleased]`
- [ ] Verified with `NIGOH_DRYRUN=1` across affected modes (`ctf`/`normal`/`fast`/`stealth`),
      checking `$?` directly rather than eyeballing output through `grep`/`head`
- [ ] Verified interrupt/resume still works (test with `set -m` in the shell) and honors all
      original flags, if state or the phase pipeline was touched
- [ ] If a new CLI flag was added: confirmed it appears in **both** `save_state()`'s persistence
      list **and** `run_chunk_child()`'s forwarded-args list — a chunked (broader-than-`/16`) scan
      exercises the second one
- [ ] Ran the full-flag combination smoke test if a new phase or opt-in flag was added:
  ```sh
  NIGOH_DRYRUN=1 NIGOH_NO_BANNER=1 nigoh -t 10.0.0.1 -m ctf -o /tmp/test \
    --paranoid --firewall-map --ot --cve --vuln --udp --badsum-test --ttl 64 --sec-headers
  ```

## How was this tested?
