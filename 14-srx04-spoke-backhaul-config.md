# srx04 — AutoVPN Spoke Configuration (Full-Tunnel Backhaul)
## Junos OS 23.2R2.21 | Role: AutoVPN Spoke

> Complete spoke configuration for the full-tunnel backhaul scenario. This spoke
> sends all non-local traffic up the tunnel to the hub via a default route on
> `st0.0`. For the design rationale see
> [`10-backhaul-design-overview.md`](10-backhaul-design-overview.md).

---

## Device Summary

| Parameter        | Value              |
|------------------|--------------------|
| Hostname         | srx04              |
| Role             | AutoVPN Spoke      |
| fxp0 (mgmt)      | 172.27.1.54/24     |
| ge-0/0/0 (WAN)   | 10.0.3.2/30        |
| ge-0/0/1 (LAN)   | 192.168.4.1/24     |
| st0.0 (tunnel)   | unnumbered (P2P)   |
| WAN peer (CSR1)  | 10.0.3.1           |
| Hub WAN IP       | 10.0.0.2           |

---

## Design Notes

srx04 is an AutoVPN spoke operating in **full-tunnel** mode: anything not local to its own
LAN (`192.168.4.0/24`) is sent up the tunnel to the hub via a default route on `st0.0`. The
hub source-NATs internet-bound traffic and hairpins spoke-to-spoke traffic. No local
internet breakout.

What makes this a full-tunnel spoke:
- Traffic-selector `remote-ip` is `0.0.0.0/0` (request everything through the tunnel).
- Default route `0.0.0.0/0 → st0.0` sends all non-local traffic to the hub.
- The host route to the hub WAN IP (`10.0.0.2/32 → 10.0.3.1`) is **critical** — it keeps
  the tunnel-carrying ESP packets out of the tunnel (anti-recursion).

Spoke-to-spoke needs no extra config here: traffic to another spoke's LAN simply follows
the default route to the hub, which routes it back out to the destination spoke.

---

## Configuration — Flat Set Format

> **Before applying:** Ensure `secrets.env` has been created with `AUTOVPN_PSK`. See `CLAUDE.md`.
> ```bash
> source secrets.env
> sed "s/\$AUTOVPN_PSK/$AUTOVPN_PSK/g" 14-srx04-spoke-backhaul-config.md > /tmp/srx04-deploy.md
> ```
> Apply from `/tmp/srx04-deploy.md`, then `rm /tmp/srx04-deploy.md`.

```
# ============================================================
# srx04 — AutoVPN Spoke (FULL-TUNNEL BACKHAUL)
# Junos OS 23.2R2.21
# Traffic Selectors — full tunnel, all non-local traffic to hub
# ============================================================

# --- System ---
set system host-name srx04
set system services ssh
set system services netconf ssh

# --- Management interface (out-of-band) ---
set interfaces fxp0 unit 0 family inet address 172.27.1.54/24

# --- WAN interface (P2P to CSR1 gi4) ---
set interfaces ge-0/0/0 description "WAN P2P to CSR1"
set interfaces ge-0/0/0 unit 0 family inet address 10.0.3.2/30

# --- LAN interface ---
set interfaces ge-0/0/1 description "LAN — Trust Zone"
set interfaces ge-0/0/1 unit 0 family inet address 192.168.4.1/24

# --- Tunnel interface (point-to-point, unnumbered) ---
set interfaces st0 unit 0 description "AutoVPN Spoke Tunnel to srx01 Hub"
set interfaces st0 unit 0 family inet

# ============================================================
# IKE Phase 1
# ============================================================

set security ike proposal AUTOVPN-IKE-PROP authentication-method pre-shared-keys
set security ike proposal AUTOVPN-IKE-PROP dh-group group14
set security ike proposal AUTOVPN-IKE-PROP authentication-algorithm sha-256
set security ike proposal AUTOVPN-IKE-PROP encryption-algorithm aes-256-cbc
set security ike proposal AUTOVPN-IKE-PROP lifetime-seconds 86400

set security ike policy AUTOVPN-IKE-POL mode main
set security ike policy AUTOVPN-IKE-POL proposals AUTOVPN-IKE-PROP
set security ike policy AUTOVPN-IKE-POL pre-shared-key ascii-text "$AUTOVPN_PSK"

# Spoke gateway — points to hub WAN IP
set security ike gateway AUTOVPN-SPOKE-GW ike-policy AUTOVPN-IKE-POL
set security ike gateway AUTOVPN-SPOKE-GW address 10.0.0.2
set security ike gateway AUTOVPN-SPOKE-GW local-identity hostname srx04.homelab.local
set security ike gateway AUTOVPN-SPOKE-GW external-interface ge-0/0/0.0
set security ike gateway AUTOVPN-SPOKE-GW version v2-only
set security ike gateway AUTOVPN-SPOKE-GW remote-identity hostname srx01.homelab.local

# ============================================================
# IPsec Phase 2
# ============================================================

set security ipsec proposal AUTOVPN-IPSEC-PROP protocol esp
set security ipsec proposal AUTOVPN-IPSEC-PROP encryption-algorithm aes-256-gcm
set security ipsec proposal AUTOVPN-IPSEC-PROP lifetime-seconds 3600

set security ipsec policy AUTOVPN-IPSEC-POL perfect-forward-secrecy keys group14
set security ipsec policy AUTOVPN-IPSEC-POL proposals AUTOVPN-IPSEC-PROP

set security ipsec vpn AUTOVPN-SPOKE bind-interface st0.0
set security ipsec vpn AUTOVPN-SPOKE ike gateway AUTOVPN-SPOKE-GW
set security ipsec vpn AUTOVPN-SPOKE ike ipsec-policy AUTOVPN-IPSEC-POL
set security ipsec vpn AUTOVPN-SPOKE establish-tunnels immediately

# Full-tunnel traffic selector: local-ip = this spoke LAN, remote-ip = 0.0.0.0/0
# (request ALL destinations — internet and other spoke LANs — through the tunnel).
set security ipsec vpn AUTOVPN-SPOKE traffic-selector TS-SPOKE local-ip 192.168.4.0/24
set security ipsec vpn AUTOVPN-SPOKE traffic-selector TS-SPOKE remote-ip 0.0.0.0/0

# TCP MSS clamping for IPsec overhead
set security flow tcp-mss ipsec-vpn mss 1350

# ============================================================
# Security Zones
# ============================================================

# untrust — WAN facing
set security zones security-zone untrust host-inbound-traffic system-services ike
set security zones security-zone untrust host-inbound-traffic system-services ping
set security zones security-zone untrust interfaces ge-0/0/0.0

# trust — LAN
set security zones security-zone trust host-inbound-traffic system-services all
set security zones security-zone trust interfaces ge-0/0/1.0

# VPN — tunnel overlay
set security zones security-zone VPN host-inbound-traffic system-services ping
set security zones security-zone VPN interfaces st0.0

# ============================================================
# Security Policies
# ============================================================

# trust → untrust: kept for completeness; effectively unused under full tunnel (no local
# breakout — internet-bound LAN traffic follows the default route into st0.0 / VPN zone)
set security policies from-zone trust to-zone untrust policy default-permit match source-address any
set security policies from-zone trust to-zone untrust policy default-permit match destination-address any
set security policies from-zone trust to-zone untrust policy default-permit match application any
set security policies from-zone trust to-zone untrust policy default-permit then permit

# trust → VPN: LAN to tunnel (internet + hub LAN + other spokes all egress here now)
set security policies from-zone trust to-zone VPN policy trust-to-vpn match source-address any
set security policies from-zone trust to-zone VPN policy trust-to-vpn match destination-address any
set security policies from-zone trust to-zone VPN policy trust-to-vpn match application any
set security policies from-zone trust to-zone VPN policy trust-to-vpn then permit

# VPN → trust: tunnel to LAN (return traffic, and inbound from another spoke via the hub)
set security policies from-zone VPN to-zone trust policy vpn-to-trust match source-address any
set security policies from-zone VPN to-zone trust policy vpn-to-trust match destination-address any
set security policies from-zone VPN to-zone trust policy vpn-to-trust match application any
set security policies from-zone VPN to-zone trust policy vpn-to-trust then permit

# ============================================================
# Routing
# ============================================================

# Anti-recursion: route to hub WAN IP via CSR1 MUST stay more-specific than the
# default below, or the tunnel-carrying ESP packets would recurse into the tunnel.
set routing-options static route 10.0.0.2/32 next-hop 10.0.3.1

# WAN underlay reachability (keeps physical/WAN traffic out of the tunnel)
set routing-options static route 10.0.0.0/30 next-hop 10.0.3.1
set routing-options static route 10.0.1.0/30 next-hop 10.0.3.1
set routing-options static route 10.0.2.0/30 next-hop 10.0.3.1
set routing-options static route 10.0.3.0/30 next-hop 10.0.3.1

# Default route into the tunnel — sends all non-local traffic to the hub via st0.0.
# CAVEAT: many vSRX images ship a management default `0.0.0.0/0 -> 172.27.1.1 via fxp0`
# in inet.0. This line would ECMP with it (traffic leaking out fxp0 instead of the
# tunnel). Move fxp0 to a management routing-instance in production, OR for lab
# validation use a specific destination route instead, e.g.:
#   set routing-options static route 10.100.100.1/32 next-hop st0.0   (sim-internet)
# See 10-backhaul-design-overview.md §6. Validated with the /32 form.
set routing-options static route 0.0.0.0/0 next-hop st0.0

# ============================================================
# Commit
# ============================================================
commit check
commit
```

---

## Verification Commands

```
# IKE / IPsec SA to hub (10.0.0.2)
show security ike security-associations
show security ipsec security-associations

# Routing — default should point at st0.0; 10.0.0.2/32 must remain via 10.0.3.1
show route 0.0.0.0/0
show route 10.0.0.2

# Internet backhaul through the hub (CSR1 loopback = simulated internet)
ping 10.100.100.1 source 192.168.4.1 count 5

# Spoke-to-spoke (e.g. to srx02), backhauled via the hub
ping 192.168.2.1 source 192.168.4.1 count 5

# IKE debug log
show log iked | last 50
```
