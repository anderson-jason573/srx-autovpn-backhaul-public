# SRX AutoVPN — Full-Tunnel Backhaul Design

A Juniper AutoVPN hub-and-spoke IPsec lab where every spoke **backhauls all
non-local traffic** up the tunnel to a central hub. The hub source-NATs
internet-bound traffic out its WAN and hairpins spoke-to-spoke traffic back
out the tunnel un-NAT'd. Built and validated on four vSRX firewalls (Junos OS
23.2R2.21) plus a Cisco IOS-XE router acting as the WAN transit / simulated
internet.

---

## 1. Topology and Addressing

```
                         ┌─────────────────────────┐
                         │  CSR1 (Cisco IOS-XE)     │
                         │  WAN transit router      │
                         │  loopback 10.100.100.1   │  ← "the internet"
                         └────────────┬────────────┘
                  10.0.0.0/30  10.0.1.0/30  10.0.2.0/30  10.0.3.0/30
                        │          │           │           │
                  ┌─────┴───┐  ┌───┴────┐  ┌───┴────┐  ┌───┴────┐
                  │ srx01   │  │ srx02  │  │ srx03  │  │ srx04  │
                  │  HUB    │  │ spoke  │  │ spoke  │  │ spoke  │
                  └────┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
              192.168.1.0/24  .2.0/24      .3.0/24     .4.0/24   (LANs)
                                  │
                   IPsec tunnels (st0.0) ride the WAN underlay;
                   all spokes terminate on the hub's single st0.0.
```

| Device | Role | Mgmt (fxp0) | WAN (ge-0/0/0) | WAN peer (CSR1) | LAN (ge-0/0/1) |
|--------|------|-------------|----------------|-----------------|----------------|
| srx01  | Hub + internet egress | 172.27.1.50 | 10.0.0.2/30 | 10.0.0.1 | 192.168.1.1/24 |
| srx02  | Spoke | 172.27.1.51 | 10.0.1.2/30 | 10.0.1.1 | 192.168.2.1/24 |
| srx03  | Spoke | 172.27.1.52 | 10.0.2.2/30 | 10.0.2.1 | 192.168.3.1/24 |
| srx04  | Spoke | 172.27.1.54 | 10.0.3.2/30 | 10.0.3.1 | 192.168.4.1/24 |
| CSR1   | WAN transit / sim-internet | 172.27.1.53 | four `/30` links | — | loopback 10.100.100.1 |

- **WAN underlay:** each SRX connects to CSR1 over a dedicated `/30`. CSR1 routes
  between all four `/30`s, so any SRX WAN IP can reach any other — this is the
  transport the IPsec tunnels ride over.
- **Management:** out-of-band on `172.27.1.0/24` via `fxp0`. Never used for tunnel
  traffic.
- **"The internet":** CSR1's loopback `10.100.100.1` stands in for an upstream
  internet destination. CSR1 has the hub's `10.0.0.0/30` connected, so it can
  return traffic to the hub's NAT address (`10.0.0.2`) with no extra config.

---

## 2. AutoVPN Fundamentals (how the tunnels form)

AutoVPN lets a single hub gateway accept connections from any number of spokes
without per-spoke configuration on the hub. The pieces:

- **IKEv2 / IPsec.** Phase 1: pre-shared key, DH group 14, SHA-256, AES-256-CBC,
  `v2-only`. Phase 2: ESP, AES-256-GCM, PFS group 14, 3600 s lifetime. TCP MSS is
  clamped to 1350 to absorb IPsec overhead.
- **Dynamic hub gateway.** The hub gateway uses `dynamic hostname homelab.local`
  with `dynamic ike-user-type group-ike-id`, so it accepts any spoke presenting an
  IKE ID under `*.homelab.local`. Each spoke sets `local-identity hostname
  srxNN.homelab.local` and the hub sets `local-identity hostname
  srx01.homelab.local`; spokes pin the hub with `remote-identity hostname
  srx01.homelab.local`.
- **Single bound tunnel interface.** Every spoke terminates on the hub's one
  `st0.0`. There is no per-spoke `st0` unit on the hub.
- **Auto Route Insertion (ARI).** When a spoke's IPsec SA comes up, the hub
  automatically installs a route to that spoke's LAN `/24` pointing at `st0.0`
  (shown as protocol `ARI-TS`). The hub needs no static routes to spoke LANs.
- **Traffic selectors** define which source/destination prefixes each end will
  carry in the tunnel. They are the lever this scenario pulls — see §5.

> **Why a traffic-selector AutoVPN and not a route-based dynamic-routing design?**
> Traffic selectors keep the spoke config self-describing (each spoke declares its
> own LAN) and let ARI handle hub-side routing automatically, which is what makes
> "drop in another spoke with no hub change" work.

> **Authentication — lab PSK vs. production PKI.** This lab authenticates IKE with a
> single shared **pre-shared key** across the hub and all spokes (kept out of git as
> `$AUTOVPN_PSK`; see §8). That keeps the focus on the AutoVPN and routing mechanics,
> but it is not production practice. In production, prefer **certificate-based (PKI)
> authentication** with a per-device identity, or at minimum strong, unique PSKs held
> in a secrets manager with rotation and the length/complexity policy enforced.

---

## 3. AutoVPN Design Rationale

The full-tunnel mechanics are covered in the sections around this one; this section
steps back to the *why* — why AutoVPN with traffic selectors at all, and how it
scales. It is design rationale that applies to any AutoVPN deployment (split tunnel or
full-tunnel backhaul), included here so the lab is self-explaining.

### 3.1 Why AutoVPN?

Juniper AutoVPN allows the hub to accept connections from an unlimited number of spoke
devices using a **single IKE gateway configuration**. When a new spoke is deployed, it
connects to the hub using the pre-configured shared PSK — the hub configuration does
not change.

### 3.2 Why Traffic Selectors Instead of OSPF/iBGP?

Traffic selectors with point-to-point st0 mode were chosen over dynamic routing
protocols (OSPF or iBGP with point-to-multipoint st0) for the following reasons:

| | OSPF/iBGP + P2MP | Traffic Selectors + ARI |
|---|---|---|
| Hub config changes for new spoke | None | None |
| Max spokes (conservative, single hub) | ~30–40 | ~15,000 (SRX5800) |
| Dynamic routing on overlay | Yes | No — not needed for hub-and-spoke |
| Spoke routing complexity | Auto via OSPF | One static default route |
| Third-party spokes supported | No | Yes |
| ARI auto-installs hub routes | N/A | Yes |

For a pure hub-and-spoke design, dynamic routing on the overlay provides no benefit —
spokes only ever need to reach the hub, and the hub only needs to reach each spoke's
LAN. Traffic selectors with ARI handle both requirements without any routing protocol
overhead, and scale to enterprise deployments with thousands of spokes.

### 3.3 Point-to-Point st0 Mode

The `st0.0` interface on all devices is in the default **point-to-point** mode (no
`multipoint` keyword). This is required when using traffic selectors —
point-to-multipoint mode does not support traffic selectors.

st0 interfaces are **unnumbered** in this design. IP addressing on the tunnel
interface is not required since routing is handled by static routes and ARI rather
than a routing protocol.

### 3.4 Scaling Considerations

| Platform | Max IPsec Tunnels (approx.) |
|---|---|
| SRX300/320 | ~512 |
| SRX340/345 | ~2,048 |
| SRX1500 | ~2,000 |
| SRX4100/4200 | ~10,000 |
| SRX4600/5800 | ~15,000 (with traffic selectors) |

For deployments approaching platform limits, dual-hub with redundancy is recommended
to eliminate the single point of failure and distribute spoke load.

### 3.5 Traffic Selector Scope: Specific Supernet vs. `0.0.0.0/0` Wildcard

> **Scope note.** This section is about the **hub's `remote-ip`** selector — which stays
> `192.168.0.0/16` even in full-tunnel backhaul (see the §5 table). That is a *different*
> axis from the `0.0.0.0/0` the backhaul design puts on the *spoke* `remote-ip` and the
> *hub* `local-ip` to pull internet-bound traffic up the tunnel. Don't conflate the two
> `0.0.0.0/0` uses.

The hub's traffic selector `remote-ip` controls which spoke prefixes the hub will accept
and — because ARI derives routes directly from the negotiated selector — which routes end
up in the hub's table. There are two valid strategies. **This lab uses the specific
supernet (`192.168.0.0/16`)**; the wildcard is documented here because it is the correct
choice for a different (and common) real-world situation.

In both cases the mechanism is the same: IKEv2 narrows the hub `remote-ip` against each
spoke's proposed `local-ip` to the **intersection**, and ARI installs that intersection as
a route via `st0.0`. With a spoke advertising `192.168.2.0/24`, both a hub `remote-ip` of
`192.168.0.0/16` and `0.0.0.0/0` negotiate down to the same clean `192.168.2.0/24` route.
The difference is entirely in what the hub is willing to accept *before* narrowing.

| | Specific supernet (`192.168.0.0/16`) | Wildcard (`0.0.0.0/0`) |
|---|---|---|
| **Use when** | Spoke LANs fall inside a summarizable block (well-designed addressing) | Spoke LANs are discontiguous / not summarizable (poorly-designed or acquired address space) |
| **Hub-side guardrail** | Yes — rejects at IKE negotiation any spoke advertising a prefix outside the summary | None — hub trusts every spoke to advertise honestly |
| **Zero-touch hub** | Mostly — but a new spoke whose LAN falls outside the summary forces you to widen the supernet (a hub change) | Fully — hub never needs editing regardless of what subnets spokes use |
| **Blast radius if a spoke is rogue/misconfigured** | Contained — a spoke cannot inject the hub LAN, another site's prefix, or a default route | Large — a spoke could advertise the hub LAN, another site's subnet, or `0.0.0.0/0`, and ARI would install it (route hijack); push the guardrail to security policies / per-spoke filtering instead |
| **Routing-table predictability** | High | Depends on spoke honesty |

**Why we default to the supernet here:** the lab's addressing is clean
(`192.168.{1,2,3,4}.0/24`), so `192.168.0.0/16` summarizes every spoke and keeps the
hub-side guardrail. It is the safer default whenever the address plan permits it.

**When the wildcard is the right call:** a customer with no summarizable scheme. Requiring
a clean supernet would break AutoVPN's zero-touch-hub promise for them — you would have to
enumerate every discontiguous prefix (or keep widening the supernet) as sites are added.
The wildcard lets the hub accept any spoke prefix without ever knowing it in advance, and
ARI still installs clean per-spoke routes via narrowing.

**Two caveats that apply regardless of which strategy is chosen:**

- **Spokes must propose their *specific* subnets, not `0.0.0.0/0`.** If a spoke also
  advertises a wildcard, the negotiated selector stays `0.0.0.0/0` and ARI has no specific
  prefix to install — the original failure mode. Keep the wildcard on the *hub* side only.
- **Overlapping spoke subnets are not solved by either selector.** A "poorly-designed
  scheme" frequently means collisions (e.g. multiple sites all using `192.168.1.0/24`).
  ARI would receive the same prefix via `st0.0` from two spokes — an ambiguous route.
  The only real fix is **NAT** (typically static/twice-NAT at each colliding spoke so it
  presents a unique prefix to the hub). This is an addressing problem, not a VPN problem.

---

## 4. Split Tunnel vs. Full-Tunnel Backhaul

A conventional AutoVPN spoke runs a **split tunnel**: only hub-LAN-bound traffic
(`remote 192.168.1.0/24`) enters the IPsec tunnel; internet traffic breaks out
locally at each spoke; spokes never talk to each other.

This design implements **full-tunnel backhaul** instead: a spoke sends
**everything that is not local to its own LAN** up the tunnel to the hub. The hub
then:

- **Source-NATs** internet-bound traffic out its WAN toward CSR1 (the simulated
  internet)
- **Hairpins** spoke-to-spoke traffic back out `st0.0` to the destination spoke
  (un-NAT'd)

Design intent, stated plainly: **if traffic is not local to a spoke, it goes to
the hub.**

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

The differences from a split tunnel are confined to four things: the traffic
selector scope, spoke routing, hub egress routing/NAT, and two added hub security
policies. Everything else — topology, addressing, IKE/IPsec parameters, the
AutoVPN dynamic-gateway mechanics — is standard AutoVPN.

---

## 5. Traffic Selectors (the core of it)

Full-tunnel is fundamentally a traffic-selector change. The spoke must *request*
the whole internet through the tunnel, and the hub must *accept* any destination.

| Device | Split tunnel | Full-tunnel backhaul |
|--------|--------------|----------------------|
| Spoke `local-ip` | `192.168.x.0/24` (its LAN) | `192.168.x.0/24` — same |
| Spoke `remote-ip` | `192.168.1.0/24` (hub LAN only) | **`0.0.0.0/0`** (everything) |
| Hub `local-ip` | `192.168.1.0/24` (hub LAN) | **`0.0.0.0/0`** (any destination) |
| Hub `remote-ip` | `192.168.0.0/16` | `192.168.0.0/16` — same |

Full tunnel **requires the wildcard by definition** — there is no summarized
supernet for "the entire internet." This is the legitimate use of `0.0.0.0/0` in a
traffic selector.

Crucially, a hub `local-ip` of `0.0.0.0/0` does **not** create a default route via
`st0.0`. IKEv2 still narrows the negotiated selector to hub `0.0.0.0/0` ∩ spoke
`192.168.x.0/24` = the spoke's specific `/24`, and **ARI still installs clean
per-spoke `/24` routes** on the hub — because ARI keys off the *remote* selector
(the spoke side), not the local one.

Spoke-to-spoke needs **no extra traffic-selector config**: with the hub `local-ip`
at `0.0.0.0/0`, the inbound SA from spoke A already accepts a destination of spoke
B's subnet, and the outbound SA to spoke B already accepts spoke A as the source.

---

## 6. Routing Changes

### Spoke

| | Split tunnel | Full-tunnel backhaul |
|---|---|---|
| Tunnel route | `192.168.1.0/24 → st0.0` (hub LAN only) | **`0.0.0.0/0 → st0.0`** (default into tunnel) |
| Hub WAN host route | `10.0.0.2/32 → <CSR /30>` | **same — and now CRITICAL** |
| WAN `/30` routes | via `<CSR /30>` | same |

**The anti-recursion route is the #1 gotcha of full tunnel.** The spoke default now
points into `st0.0`, but the ESP packets that *carry* the tunnel are destined to
the hub's WAN IP (`10.0.0.2`). If that traffic also followed the default into the
tunnel you would get infinite recursion and an instant black-hole. The specific
host route `10.0.0.2/32 → <CSR /30 next-hop>` (and the connected WAN `/30`) must
stay more-specific than the new default. Keep them.

### Hub

| | Split tunnel | Full-tunnel backhaul |
|---|---|---|
| WAN `/30` routes | via `10.0.0.1` | same |
| Spoke LAN routes | ARI `192.168.x.0/24 → st0.0` | same (ARI) |
| Default route | none | **add `0.0.0.0/0 → 10.0.0.1`** (so de-encapsulated internet traffic egresses to CSR1) |

> **Caveat — competing management default route.** The vSRX image used in this lab
> ships a **management default route** `0.0.0.0/0 → 172.27.1.1 via fxp0` in `inet.0`
> by default (not in the configs here). This collides with the `0.0.0.0/0` routing
> changes above:
>
> - Adding a second `0.0.0.0/0` (via `st0.0` on a spoke, or via `10.0.0.1` on the
>   hub) does **not** override it — Junos installs both next-hops as **ECMP**, so
>   half the traffic would wrongly egress `fxp0`, and the NAT / `untrust`-zone
>   security policy would not even apply. *Removing* the management default risks
>   cutting the out-of-band management path back to the device.
> - **Production fix:** put management (`fxp0`) in a dedicated routing-instance (a
>   management VRF) so `inet.0`'s default can legitimately point into the tunnel /
>   WAN. Then the `0.0.0.0/0` routes above are correct exactly as written.
> - **Lab shortcut (what was actually deployed and validated here):** instead of a
>   full default, route the *simulated-internet destination* specifically —
>   `10.100.100.1/32 → st0.0` on the spoke and `10.100.100.1/32 → 10.0.0.1` on the
>   hub. Each spoke also carries `192.168.0.0/16 → st0.0` so hub-LAN and
>   spoke-to-spoke traffic backhauls the same way (see §9). Being more-specific than
>   the management default, these exercise the full backhaul + NAT data path without
>   disturbing management. The config files carry the production `0.0.0.0/0` form
>   plus these specific-route alternatives as a commented note.

---

## 7. Hub NAT and Security Policy Changes

### Source NAT (internet egress only)

The hub translates spoke private sources to its own routable WAN IP so the upstream
has an address to return traffic to. PAT to the egress interface:

```
set security nat source rule-set VPN-BACKHAUL from zone VPN
set security nat source rule-set VPN-BACKHAUL to zone untrust
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match source-address 192.168.0.0/16
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET match destination-address 0.0.0.0/0
set security nat source rule-set VPN-BACKHAUL rule SNAT-INTERNET then source-nat interface
```

The rule-set is scoped `from zone VPN to zone untrust`, so it matches **only
internet egress**. Spoke-to-spoke (`VPN → VPN`) and spoke-to-hub-LAN (`VPN →
trust`) never match it and stay un-NAT'd — real private source IPs are preserved
between sites.

> Hub-LAN hosts (`192.168.1.0/24`) reaching the internet are `trust → untrust` and
> are *not* covered by this rule-set. If you want the hub to be the single NAT
> egress for its own LAN too, add a parallel `from zone trust to zone untrust`
> source-NAT rule. Out of scope for the stated requirement (spoke backhaul), so not
> included by default.

### Security policies (hub)

| Policy | Split tunnel | Full-tunnel backhaul |
|--------|--------------|----------------------|
| `trust → untrust` | permit | same |
| `trust → VPN` | permit | same |
| `VPN → trust` | permit | same |
| `VPN → untrust` | — | **add permit** (spoke internet egress) |
| `VPN → VPN` | — | **add permit** (spoke-to-spoke hairpin) |

Return traffic for both NAT'd internet flows and inter-spoke flows is handled by
the stateful session table — no reverse policy is required.

### Security policies (spoke)

No changes. `trust → VPN` (outbound) and `VPN → trust` (inbound, incl. traffic from
another spoke arriving via the hub) already permit. Under full tunnel the spoke's
`trust → untrust` policy is effectively dead (no local breakout) but is harmless to
leave.

---

## 8. Deployment Order

Deploy CSR1 first (the WAN fabric must be up before any SRX can reach its peers),
then the hub, then the spokes in any order:

```
1. CSR1   (WAN transit) → 05-csr1-config.md
2. srx01  (hub)         → 11-srx01-hub-backhaul-config.md
3. srx02  (spoke)       → 12-srx02-spoke-backhaul-config.md
4. srx03  (spoke)       → 13-srx03-spoke-backhaul-config.md
5. srx04  (spoke)       → 14-srx04-spoke-backhaul-config.md
```

The pre-shared key is kept out of the config files as a `$AUTOVPN_PSK` placeholder
and substituted at deploy time from a local `secrets.env` (see the README and
`secrets.env.example`). The PSK must be identical on the hub and every spoke.

---

## 9. Validation

This scenario has been validated end-to-end on all four SRX (hub + three spokes) on
Junos 23.2R2.21: the hub shows three IKE SAs UP, three IPsec SAs, and three clean
ARI routes (`192.168.2/3/4.0/24` via `st0.0`), with a stable data plane (no core
dumps). Both data paths pass 5/5 with 0% loss.

> Validated using the `/32` lab routing form rather than a real default route (see
> the management-default caveat in §6): on each spoke, `10.100.100.1/32 → st0.0`
> (sim-internet) and `192.168.0.0/16 → st0.0` (hub LAN + all other spoke LANs, for
> spoke-to-spoke). These stay more-specific than the image's management default and
> exercise the full backhaul + NAT + hairpin data path without disturbing
> management.

### Internet backhaul

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

### Spoke-to-spoke

```
# from srx02 LAN, reach srx03 LAN, backhauled through the hub:
ping 192.168.3.1 source 192.168.2.1 count 5       # on srx02
```

On the hub, the session should show `192.168.2.x -> 192.168.3.x` entering and
leaving on `st0.0` (zone `VPN → VPN`), **with no NAT translation** applied.

---

## 10. Caveats and Tradeoffs

1. **Anti-recursion route (see §6)** — the single most common way full tunnel
   breaks. The spoke must keep a more-specific route to the hub's WAN IP via the
   physical underlay.
2. **No local breakout — everything hairpins through the hub.** All spoke internet
   traffic concentrates on the hub's WAN link, CPU (encrypt/decrypt), and NAT table.
   This is the central capacity-planning decision in production. The upside — and
   usually the actual motivation — is **centralized egress**: all internet traffic
   can be inspected/filtered/logged at one point (UTM/IDP at the hub).
3. **MTU / MSS** — backhauling large flows over ESP makes fragmentation more likely.
   The `tcp-mss ipsec-vpn 1350` clamp is retained; lower it if PMTU issues appear
   with the added NAT.
4. **Spoke-to-spoke hairpin** rides in and out the same `st0.0` on the hub. This is
   fine in AutoVPN with traffic selectors as long as the `VPN → VPN` policy permits
   it.
5. **Lab "internet" = CSR1 loopback `10.100.100.1`.** CSR1 has the `10.0.0.0/30`
   link connected, so it can return traffic to the hub's NAT address (`10.0.0.2`)
   without extra CSR config.

---

## 11. Files in This Repository

| File | Device | Contents |
|------|--------|----------|
| `10-backhaul-design-overview.md` | All | This document |
| `11-srx01-hub-backhaul-config.md` | srx01 | Hub — full-tunnel TS, default route, source NAT, `VPN→untrust` + `VPN→VPN` policies |
| `12-srx02-spoke-backhaul-config.md` | srx02 | Spoke — full-tunnel TS, default route via `st0.0` |
| `13-srx03-spoke-backhaul-config.md` | srx03 | Spoke — full-tunnel TS, default route via `st0.0` |
| `14-srx04-spoke-backhaul-config.md` | srx04 | Spoke — full-tunnel TS, default route via `st0.0` |
| `05-csr1-config.md` | CSR1 | WAN transit router / simulated internet |
| `secrets.env.example` | — | Template for the local `secrets.env` holding `AUTOVPN_PSK` |

---

## 12. Common Commands and Troubleshooting

General-purpose Junos commands for operating and troubleshooting route-based IPsec
VPNs on SRX. These are not specific to this lab — they apply to any SRX IKE/IPsec
deployment. See the Juniper references in §13 for full command documentation.

### Tunnel status

All are operational-mode `show` commands, safe to run at any time:

| Command | What it shows |
|---------|---------------|
| `show security ike security-associations` | Phase 1 (IKE) SA state, peers, and IKE version |
| `show security ike security-associations detail` | Per-SA crypto, lifetimes, role, and identity detail |
| `show security ipsec security-associations` | Phase 2 (IPsec) SA state and the bound tunnel interface |
| `show security ipsec security-associations detail` | Proxy-IDs, protocol, lifetimes, and the SPIs per SA |
| `show security ipsec statistics` | Per-tunnel ESP encap/decap counters and error counts |
| `show security ipsec inactive-tunnels` | Tunnels that are configured but not currently established |

### Routing, policy, and NAT

| Command | What it shows |
|---------|---------------|
| `show route <prefix>` | Which next-hop / tunnel interface a destination resolves to |
| `show security flow session` | Live sessions, the in/out interfaces, and any NAT translation |
| `show security policies` / `show security match-policies` | Configured policy and which policy a given flow matches |
| `show security nat source rule all` | Source-NAT rule hit counts and translation detail |
| `show security zones` | Interface-to-zone bindings (including `st0` units) |

### Debugging IKE/IPsec negotiation

When an SA won't establish, enable IKE tracing and read the log:

```
set security ike traceoptions file ike-trace
set security ike traceoptions flag ike
set security ike traceoptions level detail
commit
```

```
show log ike-trace            # traceoptions output
clear security ike security-associations       # force Phase 1 to renegotiate
clear security ipsec security-associations     # force Phase 2 to renegotiate
```

> **IKE log daemon varies by platform/release.** On SRX platforms running the
> `iked` process (the modern IKE daemon, including recent vSRX releases) the live
> daemon log is `show log iked`; older releases use the `kmd` process and
> `show log kmd`. Configured `traceoptions` write to whatever filename you set
> regardless of daemon.

### General troubleshooting flow

| Stage | Symptom | Common causes |
|-------|---------|---------------|
| Underlay | Peers can't reach each other | Routing/firewall on the transport; verify with `ping` between WAN IPs |
| Phase 1 (IKE) | No IKE SA, or stuck negotiating | Pre-shared-key mismatch, IKE proposal/DH-group mismatch, wrong peer address or IKE identity, IKEv1-vs-IKEv2 mismatch |
| Phase 2 (IPsec) | IKE SA up but no IPsec SA | IPsec proposal/PFS mismatch, proxy-ID or traffic-selector mismatch between peers |
| Data plane | Tunnel up but traffic doesn't pass | Missing/incorrect route into the `st0` interface, security policy not permitting the flow, or `st0` in the wrong zone |
| Fragmentation | Large flows fail, small ones work | Tunnel MTU/MSS — clamp TCP MSS (`security flow tcp-mss ipsec-vpn`) to allow for ESP overhead |

---

## 13. References

- Juniper Networks — *IPsec VPN User Guide (Junos OS)*
  https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/index.html

- Juniper Networks — *AutoVPN on Hub-And-Spoke Devices*
  https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-autovpn-on-hub-and-spoke-devices.html

- Juniper Networks — *Understanding Traffic Selectors in Route-Based VPNs*
  https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-traffic-selectors-in-route-based-vpns.html

- Juniper Networks — *Example: Configuring AutoVPN with Pre-Shared Key*
  https://www.juniper.net/documentation/us/en/software/junos/interfaces-next-gen-services/topics/example/configuring-auto-vpn-pre-shared-key.html

- Juniper Networks — *IPsec VPN Configuration Overview*
  https://www.juniper.net/documentation/us/en/software/junos/vpn-ipsec/topics/topic-map/security-ipsec-vpn-configuration-overview.html

- Cisco — *IOS-XE Configuration Guide*
  https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_conn_vpnips/configuration/xe-16/sec-sec-for-vpns-w-ipsec-xe-16-book.html
