# SRX Homelab — Full-Tunnel Backhaul Scenario Design

## Proxmox Homelab Automation Project · Scenario 5 (full-tunnel backhaul)

> **Branch:** `feat/spoke-backhaul-fulltunnel`
> **Relationship to baseline:** This scenario is a variant of the AutoVPN hub-and-spoke
> design described in `00-design-overview.md` (Scenario 4). The topology, addressing,
> IKE/IPsec parameters, and AutoVPN dynamic-gateway mechanics are **unchanged**. Only the
> traffic-selector scope, spoke routing, hub egress routing/NAT, and a few security
> policies change. The baseline (split-tunnel) config files `00`–`05` remain valid and are
> not modified — deploy *either* scenario from this repo.

---

## 1. What This Scenario Changes

In the baseline (Scenario 4), spokes use a **split tunnel**: only hub-LAN-bound traffic
(`remote 192.168.1.0/24`) enters the IPsec tunnel; internet traffic breaks out locally at
each spoke; spokes never talk to each other.

This scenario implements **full-tunnel backhaul**: a spoke sends **everything that is not
local to its own LAN** up the tunnel to the hub. The hub then:

- **Source-NATs** internet-bound traffic out its WAN toward CSR1 (the simulated internet)
- **Hairpins** spoke-to-spoke traffic back out `st0.0` to the destination spoke (un-NAT'd)

Design intent, stated plainly: **if traffic is not local to a spoke, it goes to the hub.**

```
        192.168.2.10 (srx02 LAN host)
              │  dest = 10.100.100.1 (internet)  OR  192.168.3.10 (another spoke)
              ▼
        srx02  ── default route 0.0.0.0/0 via st0.0 ──► IPsec tunnel
              │
              ▼
        srx01 HUB
          ├─ internet-bound → source-NAT to 10.0.0.2 → ge-0/0/0 → CSR1
          └─ spoke-bound (192.168.3.0/24) → ARI route via st0.0 → srx03 (no NAT)
```

---

## 2. Traffic-Selector Changes (the core of it)

Full-tunnel is fundamentally a traffic-selector change. The spoke must *request* the whole
internet through the tunnel, and the hub must *accept* any destination.

| Device | Baseline (split tunnel) | This scenario (full tunnel) |
|--------|-------------------------|------------------------------|
| Spoke `local-ip` | `192.168.x.0/24` (its LAN) | `192.168.x.0/24` — unchanged |
| Spoke `remote-ip` | `192.168.1.0/24` (hub LAN only) | **`0.0.0.0/0`** (everything) |
| Hub `local-ip` | `192.168.1.0/24` (hub LAN) | **`0.0.0.0/0`** (any destination) |
| Hub `remote-ip` | `192.168.0.0/16` | `192.168.0.0/16` — unchanged |

**This scenario requires the wildcard by definition** — there is no summarized-supernet
version of "the entire internet." This is the legitimate use of `0.0.0.0/0` characterized
in `00-design-overview.md` §4.9. The wildcard was validated to narrow correctly on the
23.2R2.21 golden image: IKEv2 still narrows hub `0.0.0.0/0` ∩ spoke `192.168.x.0/24` to the
spoke's specific `/24`, and **ARI still installs clean per-spoke `/24` routes on the hub** —
changing the hub `local-ip` to `0.0.0.0/0` does *not* create a default route via `st0.0`,
because ARI keys off the *remote* selector (the spoke side), not local.

Spoke-to-spoke needs **no extra traffic-selector config**: with the hub `local-ip` at
`0.0.0.0/0`, the inbound SA from spoke A already accepts a destination of spoke B's subnet,
and the outbound SA to spoke B already accepts spoke A as the source.

---

## 3. Routing Changes

### Spoke

| | Baseline | This scenario |
|---|---|---|
| Tunnel route | `192.168.1.0/24 → st0.0` (hub LAN only) | **`0.0.0.0/0 → st0.0`** (default into tunnel) |
| Hub WAN host route | `10.0.0.2/32 → <CSR /30>` | **unchanged — and now CRITICAL** |
| WAN `/30` routes | via `<CSR /30>` | unchanged |

**The anti-recursion route is the #1 gotcha of full tunnel.** The spoke default now points
into `st0.0`, but the ESP packets that *carry* the tunnel are destined to the hub's WAN IP
(`10.0.0.2`). If that traffic also followed the default into the tunnel you would get
infinite recursion and an instant black-hole. The specific host route
`10.0.0.2/32 → <CSR /30 next-hop>` (and the connected WAN `/30`) must stay more-specific
than the new default. Keep them.

### Hub

| | Baseline | This scenario |
|---|---|---|
| WAN `/30` routes | via `10.0.0.1` | unchanged |
| Spoke LAN routes | ARI `192.168.x.0/24 → st0.0` | unchanged (ARI) |
| Default route | none | **add `0.0.0.0/0 → 10.0.0.1`** (so de-encapsulated internet traffic egresses to CSR1) |

> **Golden-image caveat (management default route).** On the 23.2R2.21 golden image used in
> this lab, `inet.0` already contains a **management default route** `0.0.0.0/0 → 172.27.1.1
> via fxp0` (baked into the image, not in our configs). This affects the `0.0.0.0/0` routing
> changes above:
>
> - Adding a second `0.0.0.0/0` (via `st0.0` on a spoke, or via `10.0.0.1` on the hub) does
>   **not** override it — Junos installs both next-hops as **ECMP**, so half the traffic
>   would wrongly egress `fxp0`, and the NAT / `untrust`-zone security policy would not even
>   apply. *Removing* the management default risks cutting the out-of-band management path
>   back to the device.
> - **Production fix:** put management (`fxp0`) in a dedicated routing-instance (a management
>   VRF) so `inet.0`'s default can legitimately point into the tunnel / WAN. Then the
>   `0.0.0.0/0` routes above are correct exactly as written.
> - **Lab validation shortcut (what was actually deployed on the pilot pair):** instead of a
>   full default, route the *simulated-internet destination* specifically —
>   `10.100.100.1/32 → st0.0` on the spoke and `10.100.100.1/32 → 10.0.0.1` on the hub. Being
>   more-specific than the management default, this exercises the full backhaul + NAT data
>   path without disturbing management. Validated 2026-06-18: 5/5, all 5 packets
>   source-NAT'd to the hub WAN IP, zero cores.

---

## 4. Hub NAT and Security Policy Changes

### Source NAT (internet egress only)

The hub translates spoke private sources to its own routable WAN IP so the upstream has an
address to return traffic to. PAT to the egress interface:

```
set security nat source rule-set VPN-BACKHAUL from zone VPN
set security nat source rule-set VPN-BACKHAUL to zone untrust
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match source-address 192.168.0.0/16
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match destination-address 0.0.0.0/0
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET then source-nat interface
```

The rule-set is scoped `from zone VPN to zone untrust`, so it matches **only internet
egress**. Spoke-to-spoke (`VPN → VPN`) and spoke-to-hub-LAN (`VPN → trust`) never match it
and stay un-NAT'd — real private source IPs are preserved between sites.

> Hub-LAN hosts (`192.168.1.0/24`) reaching the internet are `trust → untrust` and are *not*
> covered by this rule-set. If you want the hub to be the single NAT egress for its own LAN
> too, add a parallel `from zone trust to zone untrust` source-NAT rule. Out of scope for the
> stated requirement (spoke backhaul), so not included by default.

### Security policies (hub)

| Policy | Baseline | This scenario |
|--------|----------|----------------|
| `trust → untrust` | permit | unchanged |
| `trust → VPN` | permit | unchanged |
| `VPN → trust` | permit | unchanged |
| `VPN → untrust` | — | **add permit** (spoke internet egress) |
| `VPN → VPN` | — | **add permit** (spoke-to-spoke hairpin) |

Return traffic for both NAT'd internet flows and inter-spoke flows is handled by the
stateful session table — no reverse policy is required.

### Spoke security policies

No changes. `trust → VPN` (outbound) and `VPN → trust` (inbound, incl. traffic from another
spoke arriving via the hub) already permit in the baseline. Under full tunnel the spoke's
`trust → untrust` policy is effectively dead (no local breakout) but is harmless to leave.

---

## 5. Deployment Order

CSR1 (`05-csr1-config.md`) is **unchanged** — the WAN fabric is identical. Deploy in the
same order as the baseline:

```
1. srx01  (hub)   → 11-srx01-hub-backhaul-config.md
2. srx02  (spoke) → 12-srx02-spoke-backhaul-config.md
3. srx03  (spoke) → 13-srx03-spoke-backhaul-config.md
4. srx04  (spoke) → 14-srx04-spoke-backhaul-config.md
```

PSK substitution from `secrets.env` works exactly as in the baseline (see `CLAUDE.md`).

---

## 6. Validation

> **Status (2026-06-18): FULLY VALIDATED — all 4 SRX on 23.2R2.21 (hub + 3 spokes).**
> Hub shows 3 IKE SAs UP, 3 IPsec SAs, and 3 clean ARI routes (`192.168.2/3/4.0/24` via
> st0.0). Validated 5/5, 0% loss, zero cores throughout:
> - **Internet backhaul:** each spoke → `10.100.100.1` (sourced from its LAN gateway).
>   Hub source NAT `SNAT-INTERNET` = 93 translation hits / 93 successful sessions to the
>   hub WAN IP.
> - **Spoke-to-spoke:** srx02→srx03 and srx03→srx04 LAN gateways. Hub `VPN→VPN`
>   `spoke-to-spoke` policy matched (hit count rising) and the flows did **not** match the
>   `VPN→untrust` NAT rule — confirming inter-spoke traffic is hairpinned un-NAT'd.
>
> Deployed via the `/32` + `192.168.0.0/16` lab routing form, not a real default (see the
> golden-image caveat in §3): on each spoke, `10.100.100.1/32 → st0.0` (sim-internet) and
> `192.168.0.0/16 → st0.0` (hub LAN + all other spoke LANs, for spoke-to-spoke). This stays
> more-specific than the baked-in management default and exercises the full backhaul + NAT +
> hairpin data path without disturbing management.

### Internet backhaul (validatable on the current pilot pair: hub + srx02)

From a spoke LAN host, reach the simulated internet (CSR1 loopback `10.100.100.1`):

```
# from a host on the spoke LAN (e.g. 192.168.2.10), or sourced from the spoke LAN gateway:
ping 10.100.100.1 source 192.168.2.1 count 5     # on the spoke
```

On the **hub**, confirm the translation and the flow path:

```
show security flow session                       # expect 192.168.2.x -> 10.100.100.1, xlated to 10.0.0.2
show security nat source rule all                # SNAT-INTERNET hit count incrementing
show security nat source summary
show route 192.168.2.0/24                         # still ARI-TS via st0.0 (clean /24)
show security ipsec security-associations         # SA up; data plane stable, zero cores
```

### Spoke-to-spoke (VALIDATED 2026-06-18 with all 3 spokes on 23.2R2.21)

```
# from srx02 LAN, reach srx03 LAN, backhauled through the hub:
ping 192.168.3.1 source 192.168.2.1 count 5       # on srx02
```

On the hub, the session should show `192.168.2.x -> 192.168.3.x` entering and leaving on
`st0.0` (zone `VPN → VPN`), **with no NAT translation** applied.

---

## 7. Caveats and Tradeoffs

1. **Anti-recursion route (see §3)** — the single most common way full tunnel breaks. The
   spoke must keep a more-specific route to the hub's WAN IP via the physical underlay.
2. **No local breakout — everything hairpins through the hub.** All spoke internet traffic
   concentrates on the hub's WAN link, CPU (encrypt/decrypt), and NAT table. This is the
   central capacity-planning decision in production. The upside — and usually the actual
   motivation — is **centralized egress**: all internet traffic can be inspected/filtered/
   logged at one point (UTM/IDP at the hub).
3. **MTU / MSS** — backhauling large flows over ESP makes fragmentation more likely. The
   `tcp-mss ipsec-vpn 1350` clamp from the baseline is retained; lower it if PMTU issues
   appear with the added NAT.
4. **Spoke-to-spoke hairpin** rides in and out the same `st0.0` on the hub. This is fine in
   AutoVPN with traffic selectors as long as the `VPN → VPN` policy permits it.
5. **Lab "internet" = CSR1 loopback `10.100.100.1`.** CSR1 has the `10.0.0.0/30` link
   connected, so it can return traffic to the hub's NAT address (`10.0.0.2`) without extra
   CSR config.

---

## 8. Files in This Scenario

| File | Device | Contents |
|------|--------|----------|
| `10-backhaul-design-overview.md` | All | This document |
| `11-srx01-hub-backhaul-config.md` | srx01 | Hub — full-tunnel TS, default route, source NAT, VPN→untrust + VPN→VPN policies |
| `12-srx02-spoke-backhaul-config.md` | srx02 | Spoke — full-tunnel TS, default route via st0.0 |
| `13-srx03-spoke-backhaul-config.md` | srx03 | Spoke — full-tunnel TS, default route via st0.0 |
| `14-srx04-spoke-backhaul-config.md` | srx04 | Spoke — full-tunnel TS, default route via st0.0 |
| `05-csr1-config.md` | CSR1 | **Unchanged** — reuse the baseline WAN router config |
