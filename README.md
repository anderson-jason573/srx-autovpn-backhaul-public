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
  support. This lab used vSRX **23.2R2.21**.
- The `junos-ike` package is required for AutoVPN. Install it and reboot:

  ```
  request system software add optional://junos-ike.tgz
  request system reboot
  ```
- One Cisco IOS-XE router (this lab used a CSR1000v) for the WAN fabric.
- Management reachability to every device (this lab uses out-of-band `fxp0` on
  `172.27.1.0/24`).

---

## Quick start

**1. Create your PSK file**

`secrets.env` is just a plain-text file holding one variable, `AUTOVPN_PSK`, in
`NAME=value` form (no spaces around the `=`). It keeps the real key out of git so
the configs in this repo can ship with a placeholder instead. Copy the template
and put your key in it:

```bash
cp secrets.env.example secrets.env
# edit secrets.env so the line reads, for example:
#   AUTOVPN_PSK=Y0ur-Str0ng-PSK!-here
```

Use a strong key (20+ chars, mixed case, numbers, symbols), and use the **same**
key on every device — the hub and all spokes must match. Because `.gitignore`
excludes `secrets.env`, this file stays local and is never pushed.

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

First, `source secrets.env` — this reads the file and loads `AUTOVPN_PSK` into your
current shell session as an environment variable, so the next command can reference
it as `$AUTOVPN_PSK`. (It only lasts for that terminal session; re-run it if you
open a new shell.) Then `sed` rewrites the config, swapping the placeholder for your
real key and saving the result to a temporary deploy file:

```bash
source secrets.env
sed "s/\$AUTOVPN_PSK/$AUTOVPN_PSK/g" 11-srx01-hub-backhaul-config.md > /tmp/srx01-deploy.md
# apply /tmp/srx01-deploy.md to the device, then:
rm /tmp/srx01-deploy.md
```

**Reading that `sed` command:** the `s/old/new/g` syntax means *substitute* — find
`old`, replace it with `new`, and the trailing `g` makes it *global* so every match
on a line is replaced, not just the first. Here `old` is the literal placeholder
text `$AUTOVPN_PSK` (the `\$` escapes the `$` so the shell leaves it alone and `sed`
sees the literal characters), and `new` is `$AUTOVPN_PSK` *without* the backslash, so
the shell expands it to the actual key you sourced in the previous step. `sed` prints
the rewritten config to standard output, and the `>` redirects that output into the
new file `/tmp/srx01-deploy.md` instead of your screen. The original repo file is
never modified.

**Applying the deploy file to the SRX:** the deploy file now holds the complete
device config as Junos flat `set` commands. You can apply it two ways.

*Option A — manually via the CLI.* Open the deploy file, copy the block of `set`
lines, then on the device:

```
ssh admin@172.27.1.50        # the SRX's fxp0 management IP
configure                    # enter configuration mode
# paste the set commands here (or use: load set terminal, paste, then Ctrl-D)
commit check                 # validate before committing
commit                       # apply
```

`commit check` confirms the candidate config is valid without activating it; `commit`
makes it live.

*Option B — with Claude Code.* This lab is built to be driven from Claude Code using
the [junos-mcp-server](https://github.com/Juniper/junos-mcp-server) MCP, which pushes
config to each SRX over NETCONF — no manual paste. With the device listed in
`devices.json` and NETCONF enabled on the SRX (`set system services netconf ssh`),
ask Claude Code to load and commit the deploy file to the device; it handles the
`commit check` / `commit` for you. See `CLAUDE.md` for the MCP setup.

Either way, repeat for each SRX config (`12-`, `13-`, `14-`), using that device's
management IP. Delete each `/tmp/*-deploy.md` file once applied — it contains the
real key in cleartext.

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

> **Heads-up — the `0.0.0.0/0` routing caveat.** vSRX does not ship with a default
> route via `fxp0`; this lab adds one so the devices have out-of-band management
> access. The side effect is that a second `0.0.0.0/0` (into the tunnel on a spoke,
> or toward the WAN on the hub) ECMPs with that management default — traffic leaks
> out `fxp0` and NAT never applies. In a live production environment you'd put
> `fxp0` in its own management routing-instance, which isolates the management
> default and lets a `0.0.0.0/0` be used normally for data traffic. In this lab we
> instead use a specific route to `10.100.100.1/32` — a loopback on CSR1 that
> simulates the internet — so it stays more-specific than the management default and
> NAT works. See design doc §5.

---

## Security note

The pre-shared key is **never** stored in this repository. Configs carry a
`$AUTOVPN_PSK` placeholder that is substituted at deploy time from a local,
git-ignored `secrets.env`. Use a strong PSK (20+ chars, mixed case, numbers,
symbols), identical on the hub and all spokes.
