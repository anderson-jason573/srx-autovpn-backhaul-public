# srx01 — AutoVPN Hub Configuration (Full-Tunnel Backhaul)
## Junos OS 23.2R2.21 | Role: AutoVPN Hub + Internet Egress

> Complete hub configuration for the full-tunnel backhaul scenario. The hub
> terminates all spokes on a single `st0.0`, source-NATs their internet-bound
> traffic out its WAN, and hairpins spoke-to-spoke traffic. For the design
> rationale see [`10-backhaul-design-overview.md`](10-backhaul-design-overview.md).

---

## Device Summary

| Parameter        | Value              |
|------------------|--------------------|
| Hostname         | srx01              |
| Role             | AutoVPN Hub + Internet Egress |
| fxp0 (mgmt)      | 172.27.1.50/24     |
| ge-0/0/0 (WAN)   | 10.0.0.2/30        |
| ge-0/0/1 (LAN)   | 192.168.1.1/24     |
| st0.0 (tunnel)   | unnumbered (P2P)   |
| WAN peer (CSR1)  | 10.0.0.1           |
| NAT egress       | source-nat interface (PAT to 10.0.0.2) |

---

## Design Notes

srx01 is the AutoVPN hub **and** the single internet egress point for all spokes. Spokes
default-route everything into the tunnel; the hub source-NATs internet-bound traffic out
`ge-0/0/0` toward CSR1 and hairpins spoke-to-spoke traffic back out `st0.0` (un-NAT'd).

What makes this the full-tunnel hub:
- Traffic selector `local-ip` is `0.0.0.0/0` (hub accepts any destination from spokes).
  IKEv2 still narrows per-spoke and ARI still installs clean `/24` routes — the hub
  `local-ip 0.0.0.0/0` does **not** create a default route via `st0.0`.
- A default route `0.0.0.0/0 → 10.0.0.1` sends de-encapsulated internet traffic to CSR1.
- Source NAT rule-set `VPN-BACKHAUL` (zone VPN → untrust) PATs spoke sources to the WAN IP.
- Security policies `VPN → untrust` (internet egress) and `VPN → VPN` (spoke-to-spoke).

---

## Configuration — Flat Set Format

> **Before applying:** Substitute `$AUTOVPN_PSK` before pasting:
> ```bash
> source secrets.env
> sed "s/\$AUTOVPN_PSK/$AUTOVPN_PSK/g" 11-srx01-hub-backhaul-config.md > /tmp/srx01-deploy.md
> ```
> Apply from `/tmp/srx01-deploy.md`, then `rm /tmp/srx01-deploy.md`.

```
# ============================================================
# srx01 — AutoVPN Hub (FULL-TUNNEL BACKHAUL)
# Junos OS 23.2R2.21
# Traffic Selectors + ARI + source NAT — hub is the internet egress
# ============================================================

# --- System ---
set system host-name srx01
set system services ssh
set system services netconf ssh

# --- Management interface (out-of-band) ---
set interfaces fxp0 unit 0 family inet address 172.27.1.50/24

# --- WAN interface (P2P to CSR1 gi1) ---
set interfaces ge-0/0/0 description "WAN P2P to CSR1"
set interfaces ge-0/0/0 unit 0 family inet address 10.0.0.2/30

# --- LAN interface ---
set interfaces ge-0/0/1 description "LAN - Trust Zone"
set interfaces ge-0/0/1 unit 0 family inet address 192.168.1.1/24

# --- Tunnel interface (point-to-point — default, no multipoint keyword) ---
set interfaces st0 unit 0 description "AutoVPN Hub Tunnel"
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

# Hub gateway — dynamic, accepts any spoke with identity matching *.homelab.local
set security ike gateway AUTOVPN-HUB-GW ike-policy AUTOVPN-IKE-POL
set security ike gateway AUTOVPN-HUB-GW dynamic hostname homelab.local
set security ike gateway AUTOVPN-HUB-GW dynamic ike-user-type group-ike-id
set security ike gateway AUTOVPN-HUB-GW dynamic reject-duplicate-connection
set security ike gateway AUTOVPN-HUB-GW local-identity hostname srx01.homelab.local
set security ike gateway AUTOVPN-HUB-GW external-interface ge-0/0/0.0
set security ike gateway AUTOVPN-HUB-GW version v2-only

# ============================================================
# IPsec Phase 2
# ============================================================

set security ipsec proposal AUTOVPN-IPSEC-PROP protocol esp
set security ipsec proposal AUTOVPN-IPSEC-PROP encryption-algorithm aes-256-gcm
set security ipsec proposal AUTOVPN-IPSEC-PROP lifetime-seconds 3600

set security ipsec policy AUTOVPN-IPSEC-POL perfect-forward-secrecy keys group14
set security ipsec policy AUTOVPN-IPSEC-POL proposals AUTOVPN-IPSEC-PROP

# Single VPN bound to st0.0 — handles ALL spokes via traffic selector
set security ipsec vpn AUTOVPN-HUB bind-interface st0.0
set security ipsec vpn AUTOVPN-HUB ike gateway AUTOVPN-HUB-GW
set security ipsec vpn AUTOVPN-HUB ike ipsec-policy AUTOVPN-IPSEC-POL

# Full-tunnel traffic selector: hub local-ip is 0.0.0.0/0 so it accepts ANY
# destination from spokes (internet + other spoke LANs). remote-ip stays the spoke summary.
# IKEv2 narrows per-spoke; ARI still installs each specific /24 via st0.0 (local 0.0.0.0/0
# does NOT create a default route via st0.0). See 10-backhaul-design-overview.md §5.
set security ipsec vpn AUTOVPN-HUB traffic-selector TS-ALL local-ip 0.0.0.0/0
set security ipsec vpn AUTOVPN-HUB traffic-selector TS-ALL remote-ip 192.168.0.0/16

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

# trust → untrust: default permit (internet access for hub LAN)
set security policies from-zone trust to-zone untrust policy default-permit match source-address any
set security policies from-zone trust to-zone untrust policy default-permit match destination-address any
set security policies from-zone trust to-zone untrust policy default-permit match application any
set security policies from-zone trust to-zone untrust policy default-permit then permit

# trust → VPN: hub LAN to spokes
set security policies from-zone trust to-zone VPN policy trust-to-vpn match source-address any
set security policies from-zone trust to-zone VPN policy trust-to-vpn match destination-address any
set security policies from-zone trust to-zone VPN policy trust-to-vpn match application any
set security policies from-zone trust to-zone VPN policy trust-to-vpn then permit

# VPN → trust: spokes to hub LAN
set security policies from-zone VPN to-zone trust policy vpn-to-trust match source-address any
set security policies from-zone VPN to-zone trust policy vpn-to-trust match destination-address any
set security policies from-zone VPN to-zone trust policy vpn-to-trust match application any
set security policies from-zone VPN to-zone trust policy vpn-to-trust then permit

# VPN → untrust: backhauled spoke traffic out to the internet (gets source-NAT'd)
set security policies from-zone VPN to-zone untrust policy vpn-to-internet match source-address any
set security policies from-zone VPN to-zone untrust policy vpn-to-internet match destination-address any
set security policies from-zone VPN to-zone untrust policy vpn-to-internet match application any
set security policies from-zone VPN to-zone untrust policy vpn-to-internet then permit

# VPN → VPN: spoke-to-spoke hairpin through the hub (un-NAT'd, private-to-private)
set security policies from-zone VPN to-zone VPN policy spoke-to-spoke match source-address any
set security policies from-zone VPN to-zone VPN policy spoke-to-spoke match destination-address any
set security policies from-zone VPN to-zone VPN policy spoke-to-spoke match application any
set security policies from-zone VPN to-zone VPN policy spoke-to-spoke then permit

# ============================================================
# Source NAT — internet egress for backhauled spoke traffic
# ============================================================

# PAT spoke private sources to the WAN egress interface IP (10.0.0.2).
# Scoped zone VPN -> untrust, so ONLY internet egress is translated. Spoke-to-spoke
# (VPN -> VPN) and spoke-to-hub-LAN (VPN -> trust) never match and stay un-NAT'd.
set security nat source rule-set VPN-BACKHAUL from zone VPN
set security nat source rule-set VPN-BACKHAUL to zone untrust
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match source-address 192.168.0.0/16
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match destination-address 0.0.0.0/0
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET then source-nat interface

# ============================================================
# Routing
# ============================================================

# Static routes to each WAN /30 subnet via CSR1
# No static routes to spoke LANs needed — ARI installs them automatically
set routing-options static route 10.0.0.0/30 next-hop 10.0.0.1
set routing-options static route 10.0.1.0/30 next-hop 10.0.0.1
set routing-options static route 10.0.2.0/30 next-hop 10.0.0.1
set routing-options static route 10.0.3.0/30 next-hop 10.0.0.1

# Default route to the internet via CSR1 — egress path for de-encapsulated
# spoke traffic. Spoke LAN /24s remain more-specific (ARI via st0.0), so this only catches
# true internet-bound traffic.
# CAVEAT: many vSRX images ship a management default `0.0.0.0/0 -> 172.27.1.1 via fxp0`
# in inet.0. This line would ECMP with it (traffic leaking out fxp0, NAT not applying).
# Move fxp0 to a management routing-instance in production, OR for lab validation use a
# specific destination route instead, e.g.:
#   set routing-options static route 10.100.100.1/32 next-hop 10.0.0.1   (sim-internet)
# See 10-backhaul-design-overview.md §6. Validated with the /32 form.
set routing-options static route 0.0.0.0/0 next-hop 10.0.0.1

# ============================================================
# Commit
# ============================================================
commit check
commit
```

---

## Verification Commands

```
# IKE / IPsec SAs (one per spoke)
show security ike security-associations
show security ipsec security-associations

# ARI routes — still clean per-spoke /24s despite local 0.0.0.0/0
show route 192.168.2.0/24
show route 192.168.0.0/16                  # all ARI-installed spoke /24s via st0.0 (protocol ARI-TS)

# Source NAT — translations for backhauled internet traffic
show security nat source rule all
show security nat source summary
show security flow session                 # expect spoke-src -> 10.100.100.1, xlated to 10.0.0.2

# Internet reachability from the hub itself
ping 10.100.100.1 count 5

# Data-plane stability (must stay zero on 23.2R2.21)
show system core-dumps
```
