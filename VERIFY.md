# VERIFY.md: what was actually run, 2026-11-17

Environment: Windows 10 MSYS/git-bash workstation, node (Windows), git 2.x,
Docker Desktop (daemon available at session start, later stopped), no
shellcheck, no WSL. Scratch clone: `_work/_scratch/upstream` (shallow, HEAD
`1a58c90` = upstream #23). All commands were run individually and their
output inspected.

## Patch application on a fresh clean-tree clone

| check | result |
|-------|--------|
| `git apply --check scripts/generate-compose.masque.patch` | PASS (applies cleanly) |
| `git apply` + `sh -n generate-compose.sh` | PASS |
| `git apply --check keep-cold.masque.patch` | PASS (applies cleanly) |
| `git apply` + `node --check keep-cold.mjs` | PASS |

## Generator patch behaviour on the patched clone

| command | result |
|---------|--------|
| `sh ./generate-compose.sh 10` (default engine) | PASS (ran clean) |
| `git diff --exit-code docker-compose.yml` | PASS (default output byte-identical to upstream's committed file) |
| `sh ./generate-compose.sh 3 --node-engine=microwarp` | PASS |
| `docker compose config -q` (microwarp, socks) | PASS (valid compose) |
| `sh ./generate-compose.sh 3 --node-engine=microwarp --masque-proxy-mode=l4-socks` | PASS |
| `docker compose config -q` (l4-socks) | PASS (valid compose) |
| `sh ./generate-compose.sh 3 --node-engine=bogus` | PASS (exit 1, clear error) |
| `sh ./generate-compose.sh 3 --legacy-warp --node-engine=microwarp` | PASS (exit 1, mutual-exclusion error) |
| `docker-compose.microwarp.example.yml` vs freshly generated 3-node microwarp output | IDENTICAL (modulo comment header) |

Patch-apply was verified twice: once against the previously prepared trees and
once against a fresh `git clone --depth 1` made in this session after the
pre-existing `_work/upstream` turned out to be empty (not a real clone).

## Smoke check block from CONTRIBUTING

| command | result |
|---------|--------|
| `node --check server.mjs` | PASS |
| `node --check keep-cold.mjs` (patched) | PASS |
| `sh -n generate-compose.sh` | PASS |
| `sh -n setup.sh` | PASS |
| `sh -n get-status.sh` | PASS |
| `sh -n logwatch.sh` | PASS |
| `sh -n logwatch.test.sh` | PASS |
| `sh -n warp-lite/entrypoint.sh` | PASS |
| `sh -n register-node.sh` (POSIX sh parse) | PASS |
| `npm ci --no-audit --no-fund` | PASS (1 package) |
| `bash logwatch.test.sh` | PASS (**121 passed, 0 failed**, 110s) |
| `node --test test/*.test.mjs` | **98 passed / 3 failed of 101** on this Windows host |

### The 3 node-test failures: environmental, not patch-related

All three are `--sweep` tests dying in the test runner's `takeBlock` with
`port band exhausted` (port-band allocation for relay listeners; the Windows
host's dynamic-port state differs from CI). Evidence they are not caused by
either patch:

* run in isolation on the patched tree, all 3 pass: `node --test
  --test-name-pattern "sweep exits 0 when clean count meets floor|sweep exits
  1 when clean count is below floor|escalates to tier-2 when soft restart
  cannot move" test/keep-cold.test.mjs` → **3 tests, 3 pass, 0 fail**;
* the same full-suite run on **pristine upstream** (patch reverted) also
  stalls in the same port-band code path; the failures reproduce without the
  patches;
* every other keep-cold test, including all `--reset-until-clean` and
  tier-escalation tests that exercise the patched code paths, passed on the
  patched tree.

CI (Linux) is the gate for this command; the PR body notes this caveat.

## ShellCheck

Not available on this workstation (no shellcheck binary, no scoop/choco
package, WSL none, Docker daemon stopped mid-session so the
`koalaman/shellcheck` image could not be pulled). **SKIPPED, honestly**.
CI runs it; findings may surface there. The two touched/new scripts were
reviewed by hand for common sh pitfalls (quoting, POSIX constructs); no
bashisms found (`register-node.sh` passes `sh -n`).

## Privacy scan of all deliverables, `_work` and `_scratch` excluded

Programmatic scan of every `.md`/`.yml`/`.sh`/`.patch`/`.json` file for
device ids, and `device-id`-value patterns: **CLEAN**, no hits. Relay URLs
appear only as allowlist hostnames already public in upstream
(`RELAY_ALLOWED_TARGETS`), egress addresses only as `<EGRESS-IP>`
placeholders.

## Not run here: environment-bound items

* `docker compose config -q` on the remote pool server via SSH, not needed:
  the local Docker daemon validated the compose before it stopped.
* End-to-end MASQUE run: needs a TSPU-path host; prior-agent measurements
  (10/10 egress, 10m34s rebuild, 22 restarts/hour) are quoted as provided,
  not re-measured.

## Cleanup

The verification scratch clone and helper files were deleted after the run;
nothing was pushed and no GitHub actions were taken. `_work/` retains the
previous agent's prepared trees (left for reference).
