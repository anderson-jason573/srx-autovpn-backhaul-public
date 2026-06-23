# SRX AutoVPN — Full-Tunnel Backhaul

A Juniper **AutoVPN hub-and-spoke IPsec** lab where every spoke **backhauls all
non-local traffic** up the tunnel to a central hub. The hub source-NATs
internet-bound traffic out its WAN and hairpins spoke-to-spoke traffic back out
the tunnel un-NAT'd.

Built and validated on four vSRX firewalls (Junos OS **23.2R2.21**) plus a Cisco
IOS-XE router acting as the WAN transit / simulated internet.

> **Full design write-up:** [`10-backhaul-design-overview.md`](10-backhaul-design-overview.md)
> — topology, addressing, AutoVPN mechanics, traffic selectors, routing, NAT, and
> the tradeoffs of full-tunnel backhaul. Start there.

---

## What "full-tunnel backhaul" means

A conventional AutoVPN spoke runs a **split tunnel**: only hub-LAN-bound traffic
enters the tunnel, internet traffic breaks out locally, and spokes never talk to
each other. This lab does the opposite — **if traffic is not local to a spoke, it
goes to the hub:**

```
   spoke LAN host
        │  dest = internet  OR  another spoke
        ▼
   spoke ── default route 0.0.0.0/0 via st0.0 ──► IPsec tunnel
        │
        ▼
   HUB ├─ internet-bound → source-NAT out WAN → upstream
       └─ spoke-bound    → hairpin back out st0.0 → destination spoke (no NAT)
```

The motivation is **centralized egress**: all internet traffic can be inspected,
filtered, and logged at a single point (UTM/IDP on the hub).

---

## Topology

```
                    CSR1 (WAN transit, loopback 10.100.100.1 = "internet")
        10.0.0.0/30  10.0.1.0/30  10.0.2.0/30  10.0.3.0/30
              │          │            │            │
           srx01      srx02        srx03        srx04
            HUB       spoke        spoke        spoke
        192.168.1/24  .2/24        .3/24        .4/24   (LANs)
```

| Device | Role | WAN | LAN |
|--------|------|-----|-----|
| srx01  | Hub + internet egress | 10.0.0.2/30 | 192.168.1.1/24 |
| srx02  | Spoke | 10.0.1.2/30 | 192.168.2.1/24 |
| srx03  | Spoke | 10.0.2.2/30 | 192.168.3.1/24 |
| srx04  | Spoke | 10.0.3.2/30 | 192.168.4.1/24 |
| CSR1   | WAN transit / sim-internet | four `/30` links | loopback 10.100.100.1 |

---

## Repository layout

| File | Purpose |
|------|---------|
| [`10-backhaul-design-overview.md`](10-backhaul-design-overview.md) | Design write-up — read first |
| `11-srx01-hub-backhaul-config.md` | Hub config (Junos, flat `set` format) |
| `12-srx02-spoke-backhaul-config.md` | Spoke config |
| `13-srx03-spoke-backhaul-config.md` | Spoke config |
| `14-srx04-spoke-backhaul-config.md` | Spoke config |
| `05-csr1-config.md` | CSR1 WAN transit router (Cisco IOS-XE) |
| `secrets.env.example` | Template for the local `secrets.env` holding the PSK |
| `network-diagram.html` | Interactive diagram — lab topology + the backhaul traffic flow |
| `CLAUDE.md` | Deployment instructions / context for Claude Code |

---

## Prerequisites

- Four SRX (vSRX or hardware) running Junos with AutoVPN / traffic-selector
  support. This lab used vSRX **23.2R2.21**, whose golden image has `junos-ike`
  built in. On images without it, install the `junos-ike` package and reboot
  before committing AutoVPN config.
- One Cisco IOS-XE router (this lab used a CSR1000v) for the WAN fabric.
- Management reachability to every device (this lab uses out-of-band `fxp0` on
  `172.27.1.0/24`).
- On vSRX VMs: NIC driver `e1000` (for `ge-0/0/x` naming) and promiscuous mode on
  the WAN bridge so IKE/ESP isn't dropped.

---

## Quick start

**1. Create your PSK file** (never committed — see `.gitignore`):

```bash
cp secrets.env.example secrets.env
# edit secrets.env and set a strong AUTOVPN_PSK (identical on every device)
```

**2. Deploy in order** — WAN fabric first, then hub, then spokes:

```
1. CSR1   → 05-csr1-config.md
2. srx01  → 11-srx01-hub-backhaul-config.md
3. srx02  → 12-srx02-spoke-backhaul-config.md
4. srx03  → 13-srx03-spoke-backhaul-config.md
5. srx04  → 14-srx04-spoke-backhaul-config.md
```

**3. Substitute the PSK** into each SRX config before applying (the configs ship
with a `$AUTOVPN_PSK` placeholder, never the real key):

```bash
source secrets.env
sed "s/\$AUTOVPN_PSK/$AUTOVPN_PSK/g" 11-srx01-hub-backhaul-config.md > /tmp/srx01-deploy.md
# apply /tmp/srx01-deploy.md to the device, then:
rm /tmp/srx01-deploy.md
```

Each SRX config is a complete flat `set`-format block ending in `commit check` /
`commit`. CSR1 is standard IOS config applied under `conf t` and saved with `write
memory`.

---

## Validation

Validated end-to-end on all four SRX: the hub shows three IKE SAs UP, three IPsec
SAs, and three ARI-installed spoke routes (`192.168.2/3/4.0/24` via `st0.0`), with
a stable data plane.

```
# Internet backhaul — from a spoke, reach the sim-internet through the hub:
ping 10.100.100.1 source 192.168.2.1 count 5

# Spoke-to-spoke — backhauled and hairpinned through the hub (un-NAT'd):
ping 192.168.3.1 source 192.168.2.1 count 5
```

On the hub, `show security nat source rule all` shows the internet flows
translated, while spoke-to-spoke flows match the `VPN → VPN` policy with no
translation. See the design doc §8 for the full validation procedure.

> **Heads-up — the `0.0.0.0/0` routing caveat.** Many vSRX images ship a management
> default route (`0.0.0.0/0 → … via fxp0`). A second `0.0.0.0/0` into the tunnel/WAN
> will ECMP with it and break NAT. The configs include both the production
> `0.0.0.0/0` form and a more-specific `/32` lab alternative (commented). See design
> doc §5.

---

## Security note

The pre-shared key is **never** stored in this repository. Configs carry a
`$AUTOVPN_PSK` placeholder that is substituted at deploy time from a local,
git-ignored `secrets.env`. Use a strong PSK (20+ chars, mixed case, numbers,
symbols), identical on the hub and all spokes.
