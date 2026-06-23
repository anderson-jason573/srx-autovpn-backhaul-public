# CLAUDE.md — SRX AutoVPN Full-Tunnel Backhaul

Context and deployment instructions for Claude Code working in this repository.
Read the design overview before making changes:
[`10-backhaul-design-overview.md`](10-backhaul-design-overview.md).

---

## What this is

A Juniper AutoVPN hub-and-spoke IPsec lab in **full-tunnel backhaul** mode: spokes
send all non-local traffic up the tunnel to the hub, which source-NATs
internet-bound traffic out its WAN and hairpins spoke-to-spoke traffic back out the
tunnel un-NAT'd. Four vSRX (Junos **23.2R2.21**) + one Cisco IOS-XE WAN router.

This repo is **self-contained**: each config file is a complete device config, not
a diff against another scenario.

---

## Devices

| Device | Role | Mgmt (fxp0) | WAN (ge-0/0/0) | WAN peer | LAN (ge-0/0/1) |
|--------|------|-------------|----------------|----------|----------------|
| srx01  | Hub + internet egress | 172.27.1.50 | 10.0.0.2/30 | 10.0.0.1 | 192.168.1.1/24 |
| srx02  | Spoke | 172.27.1.51 | 10.0.1.2/30 | 10.0.1.1 | 192.168.2.1/24 |
| srx03  | Spoke | 172.27.1.52 | 10.0.2.2/30 | 10.0.2.1 | 192.168.3.1/24 |
| srx04  | Spoke | 172.27.1.54 | 10.0.3.2/30 | 10.0.3.1 | 192.168.4.1/24 |
| CSR1   | WAN transit / sim-internet | 172.27.1.53 | four `/30`s | — | loopback 10.100.100.1 |

---

## Repo structure

| File | Device | Format |
|------|--------|--------|
| `10-backhaul-design-overview.md` | — | Design write-up (read first) |
| `11-srx01-hub-backhaul-config.md` | srx01 | Junos flat `set` |
| `12-srx02-spoke-backhaul-config.md` | srx02 | Junos flat `set` |
| `13-srx03-spoke-backhaul-config.md` | srx03 | Junos flat `set` |
| `14-srx04-spoke-backhaul-config.md` | srx04 | Junos flat `set` |
| `05-csr1-config.md` | CSR1 | Cisco IOS |
| `secrets.env.example` | — | PSK template |

---

## The PSK is never in git

Configs carry a `$AUTOVPN_PSK` placeholder. The real key lives only in a local,
git-ignored `secrets.env` (`.gitignore` excludes `secrets.env`, deploy artifacts).

Before applying any SRX config, substitute the placeholder:

```bash
source secrets.env
sed "s/\$AUTOVPN_PSK/$AUTOVPN_PSK/g" 11-srx01-hub-backhaul-config.md > /tmp/srx01-deploy.md
# apply /tmp/srx01-deploy.md to the device, then:
rm /tmp/srx01-deploy.md
```

The PSK must be identical on the hub and every spoke (20+ chars, mixed case,
numbers, symbols). **Never commit `secrets.env`.**

---

## Deployment order

WAN fabric first, then hub, then spokes (spoke order doesn't matter):

```
1. CSR1   → 05-csr1-config.md      (conf t … end / write memory)
2. srx01  → 11-srx01-hub-backhaul-config.md
3. srx02  → 12-srx02-spoke-backhaul-config.md
4. srx03  → 13-srx03-spoke-backhaul-config.md
5. srx04  → 14-srx04-spoke-backhaul-config.md
```

Each SRX config block ends in `commit check` then `commit`.

---

## Key facts — do not deviate

- **Anti-recursion route is mandatory on every spoke.** The spoke default points
  into `st0.0`, but the ESP packets carrying the tunnel are destined to the hub WAN
  IP (`10.0.0.2`). The host route `10.0.0.2/32 → <CSR /30 next-hop>` must stay
  more-specific than the default, or the tunnel recurses into itself and
  black-holes. See design §5.
- **`0.0.0.0/0` vs. the management default.** Many vSRX images ship a management
  default `0.0.0.0/0 → 172.27.1.1 via fxp0` in `inet.0`. A second `0.0.0.0/0` (into
  the tunnel on a spoke, or toward CSR1 on the hub) **ECMPs** with it — traffic
  leaks out `fxp0` and NAT never applies. Production fix: put `fxp0` in a management
  routing-instance. Lab shortcut (validated): use a specific `10.100.100.1/32`
  route instead of a full default. Configs carry both forms (the `/32` is a
  commented alternative).
- **Hub `local-ip 0.0.0.0/0` does NOT create a default route via `st0.0`.** IKEv2
  narrows per-spoke and ARI installs clean per-spoke `/24` routes — ARI keys off the
  *remote* (spoke) selector.
- **AutoVPN hub gateway** uses `dynamic hostname homelab.local` +
  `dynamic ike-user-type group-ike-id`. Spokes set `local-identity hostname
  srxNN.homelab.local` and pin the hub with `remote-identity hostname
  srx01.homelab.local`.
- **IKE debug log is `show log iked` / `show log iked-trace`** — not `kmd` on this
  platform.

---

## vSRX environment notes

- NIC driver must be `e1000` (for `ge-0/0/x` interface naming); enable promiscuous
  mode on the WAN bridge so IKE/ESP packets aren't dropped.
- The `junos-ike` package is required for the `dynamic … group-ike-id` syntax. The
  23.2R2.21 golden image has it built in; on other images install it and reboot
  before committing AutoVPN config.
- SSH-as-root on a vSRX may land in the shell — run Junos commands via
  `cli -c "..."` from there, or use NETCONF.
- vSRX has no SFTP subsystem — transfer files with `ssh host 'cat > /tmp/f'`, not
  `scp`.
- After a same-IP VM swap, clear the stale `known_hosts` entry (the SSH host key
  changes).
- If the data plane shows 0 MB after boot, reboot the VM before configuring.

---

## Verification quick-reference

From the hub (srx01):

```
show security ike security-associations        # 3 SAs UP (10.0.1.2 / 10.0.2.2 / 10.0.3.2)
show security ipsec security-associations       # 3 SAs bound to st0.0
show route                                       # ARI routes 192.168.2/3/4.0/24 via st0.0
show security nat source rule all                # SNAT-INTERNET hits for internet egress
show system core-dumps                           # must be zero — data plane stable
```

Data-path tests (always source from a LAN gateway, or traffic-selector checks fail):

```
ping 10.100.100.1 source 192.168.2.1 count 5     # spoke → internet via hub (NAT'd)
ping 192.168.3.1  source 192.168.2.1 count 5     # spoke → spoke via hub (un-NAT'd)
```
