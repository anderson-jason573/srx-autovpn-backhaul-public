# CSR1 — Cisco IOS-XE Configuration
## WAN Transit Router | gi1 (mgmt): 172.27.1.53

---

## Device Summary

| Parameter            | Value              |
|----------------------|--------------------|
| Hostname             | CSR1               |
| Platform             | Cisco IOS-XE 17.3  |
| gi1 (mgmt / vmbr0)   | 172.27.1.53/24     |
| Loopback1 (Internet) | 10.100.100.1/32    |
| gi2 → srx01 WAN      | 10.0.0.1/30        |
| gi3 → srx02 WAN      | 10.0.1.1/30        |
| gi4 → srx03 WAN      | 10.0.2.1/30        |
| gi5 → srx04 WAN      | 10.0.3.1/30        |

---

## Role in the Design

CSR1 is a **pure WAN transit router**. It simulates the internet fabric connecting all four SRX appliances. It does **not** participate in any IPsec VPN — the SRXs build their AutoVPN tunnels across the P2P /30 links that CSR1 terminates.

CSR1's responsibilities:
- Terminate P2P /30 links to all four SRX `ge-0/0/0` interfaces
- Forward IKE and ESP traffic between the SRXs via sw1 (all on the same WAN bridge)
- Provide a simulated internet destination via `lo1` (10.100.100.1/32)

> **Note:** Because all four SRX WAN interfaces connect through the same `sw1` virtual bridge in Proxmox, CSR1 can actually route between them at layer 3. However, IKE/ESP traffic between spokes will flow SRX→CSR1→SRX since they're all on the same WAN segment — this correctly mimics internet transit behavior.

---

## Configuration

> **Before applying:** substitute the credential placeholders from `secrets.env` first.
> Replace `$CSR_USER_SECRET` **before** `$CSR_USER` — the latter is a prefix of the
> former, so order matters:
> ```bash
> source secrets.env
> sed -e "s/\$CSR_USER_SECRET/$CSR_USER_SECRET/g" \
>     -e "s/\$CSR_USER/$CSR_USER/g" \
>     05-csr1-config.md > /tmp/csr1-deploy.md
> ```
> Apply the config from `/tmp/csr1-deploy.md` (paste into the CSR console / VTY), then
> `rm /tmp/csr1-deploy.md`.

```ios
! ============================================================
! CSR1 — Cisco IOS-XE 17.3
! WAN Transit Router / Simulated Internet
! ============================================================

hostname CSR1
no aaa new-model
ip domain name enterprise.home.arpa
login on-success log

! --- Local login user for SSH (privilege 15) ---
! Credentials substituted from secrets.env at deploy time (see "Before applying").
! Cleartext secret here; IOS re-hashes it to type 9 on commit.
username $CSR_USER privilege 15 secret $CSR_USER_SECRET

! ============================================================
! Loopback — Simulated Internet destination
! ============================================================
interface Loopback1
 description Simulated Internet
 ip address 10.100.100.1 255.255.255.255
!

! ============================================================
! Management Interface (out-of-band via vmbr0)
! ============================================================
interface GigabitEthernet1
 description Management - vmbr0
 ip address 172.27.1.53 255.255.255.0
 negotiation auto
 no mop enabled
 no mop sysid
!

! ============================================================
! WAN P2P Links to SRX appliances (via sw1 WAN bridge)
! ============================================================

interface GigabitEthernet2
 description WAN P2P - srx01
 ip address 10.0.0.1 255.255.255.252
 negotiation auto
 no mop enabled
 no mop sysid
!

interface GigabitEthernet3
 description WAN P2P - srx02
 ip address 10.0.1.1 255.255.255.252
 negotiation auto
 no mop enabled
 no mop sysid
!

interface GigabitEthernet4
 description WAN P2P - srx03
 ip address 10.0.2.1 255.255.255.252
 negotiation auto
 no mop enabled
 no mop sysid
!

interface GigabitEthernet5
 description WAN P2P - srx04
 ip address 10.0.3.1 255.255.255.252
 negotiation auto
 no mop enabled
 no mop sysid
!

! ============================================================
! Forwarding / HTTP
! ============================================================
ip forward-protocol nd
ip http server
ip http authentication local
ip http secure-server
ip http client source-interface GigabitEthernet1

! ============================================================
! Routing — management default route (out-of-band reachability)
! ============================================================
ip route 0.0.0.0 0.0.0.0 172.27.1.1

! ============================================================
! SSH / Console / VTY access
! ============================================================
ip ssh time-out 60
ip ssh authentication-retries 2
ip ssh version 2
!
line con 0
 exec-timeout 0 0
 stopbits 1
line vty 0 4
 login local
 transport input ssh
!

! ============================================================
! Also present on the device (auto-generated / platform defaults,
! not operationally relevant to the lab — omitted here for brevity):
!   - self-signed PKI trustpoints + cert chains (TP-self-signed-*, SLA-TrustPoint)
!   - service call-home + CiscoTAC-1 profile
!   - license udi pid CSR1000V
!   - diagnostic bootup level minimal; spanning-tree extend system-id
! ============================================================

end
```

> **Credentials are never committed.** The `$CSR_USER` / `$CSR_USER_SECRET` placeholders
> are substituted from the local, git-ignored `secrets.env` at deploy time (see "Before
> applying" above) — the same flow the SRX configs use for the IPsec PSK. The real login
> lives only in `secrets.env` and on the device.

---

## Verification Commands

```ios
! Interface status — all WAN interfaces should be up/up
show ip interface brief

! Routing table — verify all SRX LAN routes and /30 connected routes
show ip route

! Verify connectivity to each SRX WAN IP
ping 10.0.0.2
ping 10.0.1.2
ping 10.0.2.2
ping 10.0.3.2

! Verify connectivity to simulated internet loopback
ping 10.100.100.1

! SSH session test
ssh -l admin 172.27.1.50     ! srx01 fxp0
ssh -l admin 172.27.1.51     ! srx02 fxp0
```

---

## Notes

- CSR1 **does not need any changes** when new SRX spokes are added to the AutoVPN. You only need to add a new GigabitEthernet interface for the new /30 WAN link.
- The WAN /30 subnets (`10.0.0.0/30` through `10.0.3.0/30`) all connect through sw1 in Proxmox. CSR1 is the IP gateway on each /30, so all SRX WAN traffic transits through it — including IKE negotiation between spokes and the hub.
- The simulated internet loopback (`10.100.100.1/32`) can be used in traceroute and ping tests from SRX LAN clients to validate that VPN traffic does NOT leave the overlay unencrypted.
