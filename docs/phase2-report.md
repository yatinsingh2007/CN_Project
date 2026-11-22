# Phase 2 Report: Harden, Recover and Troubleshoot

Team: Yatin Singh, Dadi Dinesh · Private domain: `app.team1.test`

Phase 2 extends the Phase 1 network; nothing was rebuilt. Mac 1 is the client and Mac 2 is the server
(DNS, nginx edge, Backend A on port 4000, Backend B on port 6000). Screenshots for every test are in
`evidence/phase2/`.

## 1. What changed from Phase 1

| Area | Phase 1 | Phase 2 |
|---|---|---|
| DNS | One resolver (dnsmasq on Mac 2) | Backup dnsmasq resolver on Mac 1 with the same records; Mac 1 lists both resolvers (primary Mac 2, backup 127.0.0.1) |
| DNS TTL | `local-ttl=30` configured | TTL behaviour demonstrated with a live record change and a manual cache flush |
| Backend access | Backend ports reachable from the LAN | pf rule on Mac 2 blocks ports 4000/6000 on `en0`; only nginx (over loopback) can reach the backends. Rule rolled back after the demo |
| Failover | `max_fails` / `fail_timeout` / `proxy_next_upstream` configured | Failover and recovery demonstrated by stopping and restarting Backend A |
| Edge | One nginx edge on Mac 2 | Standby nginx edge on Mac 1 (same certificate, same upstreams); DNS-based cutover demonstrated. Each edge adds an `X-Edge` header (`mac2` / `mac1`) so the serving edge is visible |

Because Mac 1 is a college-managed laptop whose firewall blocks incoming LAN connections, the backup
resolver and the standby edge on Mac 1 listen on its loopback address (127.0.0.1). They serve Mac 1
itself, which is the client in every test. The behaviour being tested (resolver fallback, TTL-based
cutover) is the same.

## 2. Resilience tests and results

### Extension A: Backup DNS resolver
- **Setup:** dnsmasq on Mac 1 (`listen-address=127.0.0.1`) with the same `address=` records for
  `app.team1.test` and `api.team1.test`, pointing to Mac 2. Mac 1's DNS servers: Mac 2's IP, then `127.0.0.1`.
- **Before:** names resolve through the primary resolver on Mac 2.
- **Test:** stopped the primary (`sudo brew services stop dnsmasq` on Mac 2).
- **Result:** Mac 1 still resolved `app.team1.test` to Mac 2's IP, answered by `SERVER: 127.0.0.1#53`,
  and `https://app.team1.test` kept loading. Primary restarted afterwards.
- **DNS failure vs application failure:** when DNS fails, the client cannot turn the name into an IP, so
  it never contacts the server (NXDOMAIN or a timeout). When the application fails, DNS still answers
  correctly but the server returns an error, for example nginx's `502 Bad Gateway` when both backends are down.

### Extension B: DNS TTL and controlled record change
- **Setup:** `local-ttl=30` on the project records.
- **Test:** resolved `app.team1.test` on Mac 1 with `dscacheutil` (which uses the OS cache), then changed the
  record on Mac 2 to `10.10.10.10` and restarted dnsmasq.
- **Result:** Mac 1 kept returning the old cached IP until the 30-second TTL expired, then returned
  `10.10.10.10`. After restoring the record, `dscacheutil -flushcache` + `killall -HUP mDNSResponder`
  made the change visible immediately.
- **Note:** `dig` queries the DNS server directly and bypasses the OS cache, so the cache effect is
  shown with `dscacheutil` (or curl), not dig.
- **Production relevance:** before a migration, operators lower the TTL so that a DNS change propagates
  in seconds. Clients holding a cached answer keep using the old target until their TTL expires, which
  is why old and new servers must both run during a cutover window.

### Extension C: Service isolation (backend firewall rules)
- **Rule (Mac 2, `~/iso.pf`):**
  ```
  set skip on lo0
  block drop in quick on en0 proto tcp from any to any port { 4000 6000 }
  pass all
  ```
  Original rules backed up to `~/pf.conf.backup` before loading.
- **Before:** `curl http://<Mac 2>:4000` from Mac 1 reached the backend directly.
- **After:** the same request from Mac 1 timed out, while `curl https://app.team1.test` still worked,
  because nginx reaches the backends over loopback (`127.0.0.1`), which pf skips.
- **Rollback:** `sudo pfctl -f /etc/pf.conf` restored the original rules. pf was already enabled by the
  system before the demo, so it was not disabled.
- Our backend ports are 4000/6000 instead of the brief's 3001/3002; the isolation logic is identical.

### Extension D: High-availability failover
- **Config:** `server 127.0.0.1:4000 max_fails=1 fail_timeout=10s;` (same for 6000) and
  `proxy_next_upstream error timeout http_502 http_503;`.
- **Before:** six requests from Mac 1 alternated between Backend A and Backend B.
- **Backend A stopped:** every request was served by Backend B and none failed. nginx retried the failed
  attempt on B and marked A unavailable for 10 seconds (passive health checking).
- **Backend A restarted:** after `fail_timeout` expired, requests alternated between A and B again.
- **Remaining single point of failure:** Mac 2. It runs DNS, the nginx edge and both backends, so if it
  goes down the whole service is unavailable. Removing it needs a second edge on another machine sharing
  a floating virtual IP (keepalived / VRRP) or DNS failover across two edges, plus backends on separate hosts.

### Extension E: Controlled edge migration (DNS-based cutover)
- **Setup:** standby nginx on Mac 1 with the same certificate and the same upstreams, adding
  `X-Edge: mac1`; the primary edge on Mac 2 adds `X-Edge: mac2`.
- **Test:** a loop on Mac 1 requested `https://app.team1.test` every 5 seconds and printed `X-Edge`. On
  Mac 2, the `app.team1.test` record was changed to point to the standby edge and dnsmasq restarted.
- **Result:** requests kept reaching `mac2` for up to 30 seconds (the cached TTL), then switched to
  `mac1` with no certificate errors, because both edges serve the same certificate for `app.team1.test`.
  The record was then restored to Mac 2.

## 3. Troubleshooting findings

| Symptom | Layer | Diagnosis | Fix |
|---|---|---|---|
| `dig` from Mac 1 returned NXDOMAIN from `SERVER: 2401:…` | DNS | The hotspot also hands out an IPv6 DNS server, which macOS queried first; it does not know `.test` | `networksetup -setv6off Wi-Fi` on the client during testing |
| Queries to Mac 2 timed out after changing hotspot | Network / DNS | Mac 2 had a new IP; dnsmasq was still configured for the old one | Rewrote `listen-address` / `address=` with the current IP and restarted dnsmasq |
| Port 53 on Mac 2 held by `Cloudflare` | DNS | Cloudflare WARP was running its own DNS proxy and dnsmasq was not running | Disconnected WARP, restarted dnsmasq, confirmed with `lsof -iUDP:53` |
| Backup resolver on Mac 1 never answered | DNS | Its config still contained the literal placeholder `MAC2` instead of an IP, so dnsmasq failed to start | Rewrote the config with the real IP |
| curl printed `dns:0 connect:0` | DNS | Name resolution failed before any connection; `curl -v` showed "Resolving timed out" | Tested each resolver with `dig @server` to find the broken one |

Method used throughout: check name resolution first (`dig`), then IP reachability (`ping`), then TCP
(`nc -vz host 443`), then TLS (`openssl s_client`), then HTTP (`curl -v`) and finally the nginx error log.

## 4. Learning summary

**Extension A, Backup DNS.** A second resolver keeps names working when the primary fails, but the client
only benefits if it lists both servers. We also saw that a DNS outage looks different from an application
outage: no IP at all, versus an IP that answers with an error.

**Extension B, TTL.** TTL is the trade-off between performance and agility. Our 30-second TTL meant a
record change took up to 30 seconds to reach a client unless its cache was flushed, which is exactly the
delay operators plan around when they move a service.

**Extension C, Service isolation.** Putting the backends behind nginx is not enough if their ports stay
open on the LAN. A pf rule that blocks the ports on the network interface but skips loopback enforces
"edge only" access. Keeping a backup of the original rules made the change safe to undo.

**Extension D, HA failover.** nginx's passive health checks (`max_fails`, `fail_timeout`,
`proxy_next_upstream`) hide a backend failure from users and restore balancing automatically. Redundant
backends do not remove the bigger single point of failure: the edge machine itself.

**Extension E, Edge cutover.** DNS can move all traffic to a new edge without touching clients, but not
instantly: cached answers keep sending clients to the old edge until the TTL expires, so both edges must
stay up during the switch. Serving the same certificate on both edges made the cutover invisible at the TLS layer.

**Extension F, Troubleshooting.** Most of the faults we hit were at the DNS layer, but they looked like
"the site is down". Checking each layer in order (DNS, IP, TCP, TLS, HTTP) found every cause quickly and
avoided changing things that were already working.
