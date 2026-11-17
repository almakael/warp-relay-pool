# 08 — MASQUE node engine (optional)

An alternative node engine: the relay app still dials `warp-N:1080` as an
unauthenticated SOCKS5 proxy, but the node speaks MASQUE (CONNECT-IP over
HTTP/3 on :443) instead of WireGuard UDP. Same topology, same relay contract,
same ports — only the node container changes.

Default stays `warp-lite`. Nothing in this document applies unless
`generate-compose.sh` is run with `--node-engine=microwarp`.

## When to pick it

Pick MASQUE when WireGuard cannot establish at all from the pool host. Measured
on a RU host behind a TSPU path: `warp-lite` containers start, pass their own
health check, and carry no traffic — WireGuard's UDP/2408 handshake is dropped
while TCP/443 to the same anycast ranges passes. `warp=off` on every probe;
the pool is useless even though every container reports healthy.

MASQUE/QUIC on :443 fails in the other direction: it looks like ordinary HTTPS,
so it survives paths that filter UDP and paths that SNI-filter Cloudflare's
registration API (see [Zero-state bootstrap](#zero-state-bootstrap-node-1)).

## How a node works

`ghcr.io/ccbkkb/microwarp:latest` runs [usque], which:

1. reads the node's device config from `/etc/wireguard/masque-config.json`
   (bind-mounted read-only from `state/warp-<N>/usque/config.json`);
2. opens a QUIC/HTTP-3 connection to a Cloudflare MASQUE endpoint on :443 and
   requests a CONNECT-IP tunnel (a TUN interface inside the container);
3. exposes that tunnel as a SOCKS5 listener on `0.0.0.0:1080`
   (`ALLOW_NO_AUTH=1`, `BIND_PORT=1080`).

Egress probes must go **through** that listener:

```sh
docker exec warp-relay-warp-<N> curl -s -x socks5h://127.0.0.1:1080 \
  https://www.cloudflare.com/cdn-cgi/trace
```

A plain `curl` inside the container leaves without the tunnel and reports the
host address with `warp=off`. This is the single most common mistake when
porting tooling from `warp-lite`, where the tunnel is the container's default
route.

No `/etc/wireguard/wg0.conf` exists on these nodes, and there is no WireGuard
re-key to trigger. Rotation is a container restart (see below).

### `MASQUE_PROXY_MODE`: `socks` vs `l4-socks`

| mode | listener | privileges | IPv4 egress | IPv6 egress |
|------|----------|------------|-------------|-------------|
| `socks` (default) | usque SOCKS5, userspace | none | per-device | per-device /128 |
| `l4-socks` | TUN-based L4 TCP proxy | `NET_ADMIN` | shared Cloudflare exits | per-device |

Measured on a 10-node pool: with `socks`, all 10 nodes reported **distinct**
IPv6 addresses — 10/10 unique `<EGRESS-IP>`, no two in the same /64. With
`l4-socks`, IPv4 traffic left through only **4 distinct shared** Cloudflare
egress addresses across the same 10 nodes.

Consequence: `l4-socks` collapses the pool's whole point (one throttling
domain per node) onto 4 shared domains, and 2-3 nodes always rotate into a
sibling's address. Use it only if a target requires L4 TCP semantics that
SOCKS5 cannot express; otherwise keep the default `socks`.

## Footprint

| engine | image | resident | privileges |
|--------|-------|----------|------------|
| `warp-lite` | built locally | ~1.8 MiB | `NET_ADMIN` + sysctls |
| `microwarp` (MASQUE) | `ghcr.io/ccbkkb/microwarp:latest` | 5-7 MiB (docker stats 7.1-12.0 MiB/node incl. entrypoint) | none in `socks` mode |
| `--legacy-warp` | `caomingjun/warp:latest` | ~120 MiB | `NET_ADMIN` |

10 MASQUE nodes fit in the RAM of a single legacy-WARP container. The compose
entry caps them at `mem_limit: 48M` with `GOMEMLIMIT=256MiB` inside usque.

## Registration model

One device config per node, at `state/warp-<N>/usque/config.json`: the usque
config produced by `usque register` (private key, device id, access token,
interface addresses, MASQUE endpoint). It is written once and then **reused
forever**:

* A container restart, a `docker compose up` and a host reboot all re-read the
  same file. No re-registration happens, so the registration endpoint's
  rate limit is never touched during normal operation or rotation.
* `docker restart` is the rotation primitive, not re-registration.
* The file holds a private key: it is `chmod 600`, `state/` is gitignored, and
  it must never be pasted into an issue.
* Cloudflare caps devices per account. `register-node.sh --force` overwrites a
  config but leaves the **old** device registered server-side until it is
  removed (`usque account --delete` or the dashboard). Prefer
  `--print-registration`/`--import-registration` into an empty slot over
  blind re-registration.

## Zero-state bootstrap (node 1)

`register-node.sh` needs a SOCKS5 path to `api.cloudflareclient.com` for the
registration POST. At zero state there is no node to borrow, and on some host
paths the API is SNI-filtered — direct registration fails there. Three tiers,
first one that applies wins:

1. **auto (in-pool).** The script lists running `warp-relay-warp-*` containers,
   health-checks their SOCKS port, and carries the POST through the first
   healthy node (`HTTPS_PROXY=socks5://warp-relay-warp-<M>:1080`) on the
   compose network. Covers every re-registration and node N>=2 on a fresh
   host.
2. **zero-state (node 1), via the operator workstation.**
   `./register-node.sh --print-registration <N>` prints the exact
   `docker run ... usque register` command to execute on the workstation the
   operator SSHes from — a machine that always exists and whose path to the API
   is not the pool host's path. The resulting `config.json` is brought back and
   installed with `./register-node.sh --import-registration <N> config.json`,
   which validates the usque keys, writes it `0600`, recreates the node and
   probes egress.
3. **explicit proxy (CI/automation).** `--bootstrap <socks-url>` points the
   POST at any reachable SOCKS5 proxy.

The whole flow needs curl-free, tool-free nothing on the pool host: docker, the
script, and one workstation command.

## Egress rotation

Measured: a plain `docker restart warp-relay-warp-<N>` yields a **new egress
/64** — usque re-runs CONNECT-IP and Cloudflare hands out a different prefix.
Repeated restarts do not converge back to the old prefix.

`keep-cold.mjs` must therefore treat these nodes as restart-only: there is no
`wg0.conf` to delete, and deleting `masque-config.json` would destroy the
device registration. Unmodified, its tier-2 "full reset" path deletes a
non-existent file, restarts, and retries forever: **22 restarts/hour/node**
measured, no egress change beyond the first restart. The fix is proposed
separately — see `keep-cold.masque.patch` and the bug report in the PR that
ships this document.

## After a reboot: everything comes back on its own

No human step, no re-registration, no re-key:

| piece | state | why it recovers |
|-------|-------|-----------------|
| docker | `systemctl enable docker` | starts on boot |
| node containers | `restart: always` | started by docker, re-reading their on-disk config |
| device configs | `state/warp-<N>/usque/config.json` | persisted on disk, reused, never re-registered |
| relay app | `restart: always` | started by docker, reconnects to `warp-N:1080` |
| pool systemd unit | enabled | starts with the host |
| `warp-keep-cold.timer` | enabled | probes and rotates from the first tick |

The pool comes back healthy with no operator action; the only thing a reboot
costs is a fresh egress prefix per node (harmless — see rotation above).

## Measured results (2026-11-16, 10-node pool)

| what | result |
|------|--------|
| WireGuard engine on the TSPU path | 10/10 containers healthy, 0/10 carrying traffic (`warp=off`) |
| MASQUE `warp=on` | 10/10 |
| distinct egress | 10/10 unique `<EGRESS-IP>` (no shared /64) |
| zero-state rebuild, node 1 via workstation bootstrap + nodes 2-10 auto | 10m34s wall, 10/10 registered |
| unmodified `keep-cold` on MASQUE nodes | 22 restarts/hour/node, tier-2 loop, no recovery |
| end-to-end through the relay | HTTP 200 from the allowlisted target |

## Known limitations

* **Free-tier 429 warmup.** With this pool's probe target, a *fresh* egress
  prefix gets `429` + `FreeUsageLimitError` with a retry-after of hours
  (observed ~7h, per-egress free-tier throttle) before it serves traffic.
  External to this engine; it is the same with any engine that rotates egress.
* **The image is third-party.** `ghcr.io/ccbkkb/microwarp` is pinned by tag,
  not digest. Building it locally or pinning a digest is left to the operator.
* **Node 1 on an SNI-filtered host** needs the workstation step above; there is
  no way to bootstrap the first device from a host that cannot reach the
  registration API.
* **IPv6-only rotation on `socks`.** Rotation changes the IPv6 prefix; the
  shared IPv4 anycast exit does not move (same as `warp-lite` with WARP).

[usque]: https://github.com/Diniboy1123/usque
