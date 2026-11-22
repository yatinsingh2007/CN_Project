# Private Network Service Platform

A fully local private service network built on two macOS laptops for our **Computer Networks** course project. There is no cloud and no pre-configured server: DNS, the load balancer, HTTPS and the backends are all set up by hand on our own machines.

> **Core principle:** the application stays simple, the network is the project.

![Topology](docs/topology.png)

## What it does

From the client Mac, opening **`https://app.team1.test`**:

1. **Resolves** the private name through our own DNS server (dnsmasq)
2. **Connects** over TCP to our nginx edge server
3. **Negotiates TLS** with a certificate signed by our own local CA (no browser warning)
4. **Load-balances** the request across two backend servers (round-robin)
5. **Returns** `hello from backend A` or `hello from backend B`

Every step is captured and explained with Wireshark, `dig`, `curl` and browser DevTools.

## Setup

| Machine | Role | Runs | Ports |
|---|---|---|---|
| **Mac 1** | Client | Browser, curl, dig, Wireshark | — |
| **Mac 2** | Server | dnsmasq · nginx · Backend A · Backend B | 53/UDP · 443, 80/TCP · 4000 · 6000 |
| **Mac 1** *(Phase 2)* | Backup DNS · standby edge | dnsmasq · nginx on loopback | 127.0.0.1:53 · 127.0.0.1:443 |
| **Mac 2** *(Phase 2)* | Isolation | pf rule on `en0` | blocks 4000 · 6000 from the LAN |

Mac 1 is a college-managed laptop whose firewall blocks incoming connections, so all server roles run on Mac 2. See [docs/architecture.md](docs/architecture.md#why-all-server-roles-are-on-mac-2) for how we diagnosed this.

## Tech stack

- **DNS:** dnsmasq, private `.test` domain (`app.team1.test`, `api.team1.test`)
- **Edge:** nginx: reverse proxy, round-robin load balancing, passive failover, TLS termination
- **TLS:** OpenSSL, self-generated local CA with SAN certificate
- **Backends:** Node.js + Express
- **Firewall (Phase 2):** macOS pf
- **Analysis:** Wireshark, tcpdump, `dig`, `curl`, browser DevTools

## Repository structure

```
CN_Project/
├── backendA/                 Express server, port 4000
├── backendB/                 Express server, port 6000
├── configs/
│   ├── dnsmasq/dnsmasq.conf  Private DNS records and forwarders
│   └── nginx/team1.conf      HTTPS, load balancer, redirect
├── certs/                    Public CA + server certificate, SAN config (private keys are git-ignored)
├── docs/
│   ├── architecture.md       Roles, ports, request flow by layer, design decisions
│   └── topology.png
└── evidence/
    ├── phase1/
    │   ├── Phase1_Report_IEEE.pdf   Phase 1 report (IEEE format)
    │   ├── A-network/            IP / subnet / gateway / MAC, ping both ways
    │   ├── B-dns/                dnsmasq config, dig & nslookup from the client, query log
    │   ├── C-backends/           Both backends running
    │   ├── D-loadbalancer/       nginx config, round-robin A/B
    │   ├── E-https/              Certificate, curl -v verify ok, browser padlock, redirect
    │   ├── F-caching/            Cache-Control, ETag, 304, browser cache behaviour
    │   ├── G-wireshark/          Full DNS → TCP → TLS → HTTP capture (.pcapng + screenshots)
    │   └── failures/             Five failure demos + failures.md
    └── Phase-2/
        ├── Phase2_Report_IEEE.pdf   Phase 2 report (IEEE format)
        ├── A-*.jpeg              Backup DNS answering from 127.0.0.1
        ├── B-*.jpeg              TTL caching and cache flush
        ├── C-*.jpeg              pf isolation: before, rules, after, rollback
        ├── D-*.jpeg              Failover: both backends, then Backend A down
        ├── E-*.jpeg              Standby edge and DNS cutover log
        └── report-src/           LaTeX source of the Phase 2 report
```

## Running it (on Mac 2)

**Prerequisites:** Homebrew, Node.js, `brew install dnsmasq nginx openssl@3`

1. **Backends:** in two terminals:
   ```bash
   cd backendA && npm install && npm start      # port 4000
   cd backendB && npm install && npm start      # port 6000
   ```
2. **DNS:** copy `configs/dnsmasq/dnsmasq.conf` to `/opt/homebrew/etc/dnsmasq.conf`, replace the IP with Mac 2's current Wi-Fi IP (`ipconfig getifaddr en0`), then:
   ```bash
   sudo brew services start dnsmasq
   sudo networksetup -setdnsservers Wi-Fi 127.0.0.1
   ```
3. **Certificates:** generate the CA and server certificate (commands in [certs/make-certs.md](certs/make-certs.md)), copy the `.crt` and `.key` to `/opt/homebrew/etc/nginx/certs/`.
4. **nginx:** copy `configs/nginx/team1.conf` to `/opt/homebrew/etc/nginx/servers/`, then:
   ```bash
   sudo nginx -t && sudo nginx
   ```

**On Mac 1 (client):**
```bash
sudo networksetup -setdnsservers Wi-Fi <Mac 2 IP>
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ca.crt
curl -v https://app.team1.test
```

When finished, restore normal DNS on both Macs: `sudo networksetup -setdnsservers Wi-Fi empty`.

## Endpoints

| Path | Response | Caching |
|---|---|---|
| `GET /` | `hello from backend A` / `B` | — |
| `GET /api/data` | `{"items": ["alpha", "beta", "gamma"]}` | `Cache-Control: max-age=60`, ETag, 304 support |

## Project phases

- **Phase 1 – Build & Observe** ✅ LAN, private DNS, backends, nginx load balancing, HTTPS, HTTP caching, packet-capture evidence, failure demos.
- **Phase 2 – Harden & Recover** ✅ backup DNS, TTL behaviour, backend firewall isolation, failover, DNS-based edge cutover, troubleshooting.

## Phase 2 – Harden & Recover

| Ext. | What we built | How we showed it | Evidence |
|---|---|---|---|
| **A** Backup DNS | Second dnsmasq on Mac 1 (`127.0.0.1`), same records, `local-ttl=30` | Primary stopped → `dig` answered by `127.0.0.1#53`, site still loads | [A-backup-works](evidence/Phase-2/A-backup-works.jpeg) |
| **B** TTL & record change | 30 s TTL on project records | Record changed on Mac 2; client keeps the cached IP until TTL expires; flush clears it | [B-ttl](evidence/Phase-2/B-ttl.jpeg), [B-flush](evidence/Phase-2/B-flush.jpeg) |
| **C** Service isolation | pf: `block drop in quick on en0 … port { 4000 6000 }`, `set skip on lo0` | Direct `:4000` times out; `https://app.team1.test` still works; rolled back with `pfctl -f /etc/pf.conf` | [before](evidence/Phase-2/C-before.jpeg), [rules](evidence/Phase-2/C-rules.jpeg), [after](evidence/Phase-2/C-after.jpeg), [rollback](evidence/Phase-2/C-rollback.jpeg) |
| **D** HA failover | nginx `max_fails=1 fail_timeout=10s` + `proxy_next_upstream` | Backend A stopped → every request served by B, no errors | [D-before](evidence/Phase-2/D-before.jpeg), [D-a-down](evidence/Phase-2/D-a-down.jpeg) |
| **E** Edge cutover | Standby nginx on Mac 1 with the same certificate, `X-Edge` header on both edges | DNS record moved to standby → responses switch `X-Edge: mac2` → `mac1` after the TTL, no failures or cert warnings | [standby](evidence/Phase-2/E-standby-ready.jpeg), [cutover](evidence/Phase-2/e-cutover.jpeg) |
| **F** Troubleshooting | Layer-by-layer method: `dig` → `ping` → `nc -vz` → `openssl s_client` → `curl -v` → nginx error log | Faults met during Phase 2 and their fixes are in the report | [report §VIII](evidence/Phase-2/Phase2_Report_IEEE.pdf) |

**Single point of failure:** Mac 2 still hosts the primary DNS, the primary edge and both backends. Removing it would need a second edge sharing a floating IP (VRRP/keepalived) or DNS failover, with backends on separate hosts.

Because Mac 1 cannot accept LAN connections, the backup resolver and standby edge listen on its loopback and serve Mac 1 itself (the client in every test).

## Reports

- [Phase 1 report (IEEE format)](evidence/phase1/Phase1_Report_IEEE.pdf)
- [Phase 2 report (IEEE format)](evidence/Phase-2/Phase2_Report_IEEE.pdf)

## Team

| Member | Machine | Responsibilities |
|---|---|---|
| Yatin Singh | Mac 2 (server) | DNS, nginx, TLS, backends |
| Dadi Dinesh | Mac 1 (client) | Client setup, Wireshark capture, testing, backup DNS and standby edge (Phase 2) |
