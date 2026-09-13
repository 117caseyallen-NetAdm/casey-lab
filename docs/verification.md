# Verification — Command Output From the Running Fabric

Everything in this repository's README makes claims about how the lab behaves.
This document is the evidence for those claims, captured from the live devices.

Output is verbatim, including prompts. One substitution: the firewalls' outside
addresses are replaced with `[edge-west]` / `[edge-east]`. Where output is cut,
the cut is marked `[...]`.

Captured **2026-09-13**, immediately after the NTP hierarchy was built, so the
device clocks are synchronised and the timestamps are comparable.

**Both sites share one shelf and one internet uplink.** The tunnel runs between
the two firewalls' outside interfaces across the home router's LAN, which stands
in for a WAN. The routing, the tunnel, and the vendor behaviour are real; the
distance is not.

**Devices:** PA-440 (PAN-OS) · SRX345 (Junos 20.2R3) · 2× Catalyst 3560CG
(IOS 15.0) · Catalyst 2940 (IOS 12.1, 2003) · Arista 710P (EOS 4.30.4M).

---

## Contents

1. [Two sites, one OSPF area 0](#1-two-sites-one-ospf-area-0)
2. [Both firewalls inject a default route as a Type-5 LSA](#2-both-firewalls-inject-a-default-route-as-a-type-5-lsa)
3. [The VPN pool is redistributed into OSPF and reachable from the far site](#3-the-vpn-pool-is-redistributed-into-ospf-and-reachable-from-the-far-site)
4. [The IPsec tunnel is up — and both ends agree on the SAs](#4-the-ipsec-tunnel-is-up--and-both-ends-agree-on-the-sas)
5. [Link aggregation matched to what each device supports](#5-link-aggregation-matched-to-what-each-device-supports)
6. [One time hierarchy across the fabric](#6-one-time-hierarchy-across-the-fabric) — and the one device that isn't in it
7. [What is deliberately *not* here](#7-what-is-deliberately-not-here)

---

## 1. Two sites, one OSPF area 0

**Claim:** the two sites form a single OSPF routing domain — flat area 0 — with
adjacencies running across the IPsec tunnel, not just inside each site.

**SRX345-LAB (EAST) — two adjacencies, one local, one across the tunnel:**

```
Admin@SRX345-LAB> show ospf neighbor
Address          Interface              State           ID               Pri  Dead
10.255.10.2      ae0.0                  Full            10.255.0.3         1    34
10.255.255.1     st0.0                  Full            10.255.0.1         1    31
```

`ae0.0` is the LACP bundle down to the local distribution switch. **`st0.0` is
the IPsec tunnel interface** — that second adjacency is with the PA-440 at the
far site, formed over the tunnel.

**PA440-LAB (WEST) — the mirror image:**

```
admin@PA440-LAB> show routing protocol ospf neighbor
  neighbor address:              10.255.20.2
  status:                        full
  neighbor router ID:            10.255.0.4
  area id:                       0.0.0.0
  ==========
  neighbor address:              10.255.255.2
  status:                        full
  neighbor router ID:            10.255.0.2
  area id:                       0.0.0.0
```

**Both distribution switches:**

```
LAB-CISCO-3560CG-1#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.2        0   FULL/  -        00:00:36    10.255.10.1     Port-channel1

LAB-CISCO-3560CG-2#show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.1        0   FULL/  -        00:00:37    10.255.20.1     Port-channel1
```

### What to notice

- **`area id: 0.0.0.0` on every adjacency.** One area, no ABRs, no summarisation
  — deliberately flat for a fabric this size.
- **Four OSPF routers**, identified by router ID: `10.255.0.1` (PA-440),
  `10.255.0.2` (SRX345), `10.255.0.3` (3560CG-1), `10.255.0.4` (3560CG-2).
- **`FULL/  -` on the Cisco output.** The `-` where a DR/BDR role would appear
  means no designated router is elected — the port-channels are configured
  `ip ospf network point-to-point`.
- **The Arista is not in this list, by design.** It is an access-layer switch
  with a static default route (`ip route 0.0.0.0/0 10.99.10.1`), not an OSPF
  speaker. `show ip ospf neighbor` on it returns nothing — expected, not a
  fault. Four routers run OSPF; the fifth switch does layer 2 and a default.

---

## 2. Both firewalls inject a default route as a Type-5 LSA

**Claim:** each site's firewall advertises a default route into OSPF, so
internal devices learn their way out without static configuration.

```
LAB-CISCO-3560CG-1#show ip ospf database external

            OSPF Router with ID (10.255.0.3) (Process ID 1)

                Type-5 AS External Link States

  LS age: 929
  Options: (No TOS-capability, No DC)
  LS Type: AS External Link
  Link State ID: 0.0.0.0 (External Network Number )
  Advertising Router: 10.255.0.1
  LS Seq Number: 8000004C
  Checksum: 0xCC26
  Length: 36
  Network Mask: /0
        Metric Type: 1 (Comparable directly to link state metric)
        MTID: 0
        Metric: 10
        Forward Address: 0.0.0.0
        External Route Tag: 0

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 744
  Options: (No TOS-capability, DC)
  LS Type: AS External Link
  Link State ID: 0.0.0.0 (External Network Number )
  Advertising Router: 10.255.0.2
  LS Seq Number: 8000002E
  Checksum: 0x21CE
  Length: 36
  Network Mask: /0
        Metric Type: 1 (Comparable directly to link state metric)
        MTID: 0
        Metric: 10
        Forward Address: 0.0.0.0
        External Route Tag: 0

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 1091
  Options: (No TOS-capability, DC)
  LS Type: AS External Link
  Link State ID: 10.50.1.0 (External Network Number )
  Advertising Router: 10.255.0.4
  LS Seq Number: 80000044
  Checksum: 0x70B
  Length: 36
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        MTID: 0
        Metric: 20
        Forward Address: 0.0.0.0
        External Route Tag: 0
```

### What to notice

- **Two separate `0.0.0.0/0` LSAs**, one from `10.255.0.1` (PA-440) and one from
  `10.255.0.2` (SRX345). Each site has its own exit, and every router sees both.
- **A third LSA for `10.50.1.0/24`** from `10.255.0.4`. That is the VPN pool,
  and section 3 follows it from origin to far-site route table.
- **`Metric Type: 1`** on the defaults. E1 folds the cost of reaching the ASBR
  into the metric, so each router prefers its local firewall and keeps the far
  one as a backup across the tunnel. E2 would give the same answer here, but only
  because two E2 routes with equal external metric tie-break on the cost to the
  ASBR — E1 keeps the local preference even if the two firewalls advertised
  different metrics.
- **`Forward Address: 0.0.0.0`** means "send traffic to the advertising router
  itself" rather than to some third-party next hop.

---

## 3. The VPN pool is redistributed into OSPF and reachable from the far site

**Claim:** the WireGuard client pool (`10.50.1.0/24`) is routed rather than
NATed, redistributed into OSPF, and therefore reachable fleet-wide — including
from the opposite site, across the tunnel.

This is the claim the whole routed-not-NATed design exists to support, so here
is the full chain: origin → advertisement → far-side installation.

**Origin — 3560CG-2 (WEST), where the pool lives:**

```
LAB-CISCO-3560CG-2#show ip route 10.50.1.0
Routing entry for 10.50.1.0/24
  Known via "static", distance 1, metric 0
  Redistributing via ospf 1
  Advertised by ospf 1 subnets route-map STATIC-TO-OSPF
  Routing Descriptor Blocks:
  * 10.20.2.50
      Route metric is 0, traffic share count is 1
```

**Advertisement — the third LSA in the section 2 output above, repeated here:**

```
  Link State ID: 10.50.1.0 (External Network Number )
  Advertising Router: 10.255.0.4
  Network Mask: /24
        Metric Type: 2 (Larger than any link state path)
        Metric: 20
```

Note *where* that came from: it is in the link-state database of **3560CG-1, at
the far site**, advertised by `10.255.0.4` — the WEST switch — having crossed the
tunnel.

**Installation — 3560CG-1 at the FAR site (EAST):**

```
LAB-CISCO-3560CG-1#show ip route 10.50.1.0
Routing entry for 10.50.1.0/24
  Known via "ospf 1", distance 110, metric 20, type extern 2, forward metric 12
  Last update from 10.255.10.1 on Port-channel1, 1d13h ago
  Routing Descriptor Blocks:
  * 10.255.10.1, from 10.255.0.4, 1d13h ago, via Port-channel1
      Route metric is 20, traffic share count is 1
```

### What to notice

- **`Known via "static"` on the origin, `Known via "ospf 1"` at the far site.**
  Same prefix, two different origins — that *is* redistribution, visible.
- **`route-map STATIC-TO-OSPF`.** The static route is not redistributed blindly;
  a prefix-list inside the route-map restricts redistribution to this pool
  specifically. Redistributing every static route into OSPF is how people leak
  things they did not intend to.
- **`* 10.20.2.50`** — the next hop is the WireGuard container, on the WEST data
  VLAN. Traffic to a VPN client is routed to that host, not translated.
- **`type extern 2`** on the far side, matching `Metric Type: 2` in the LSA. The
  external cost stays at 20 regardless of how far a router is from the ASBR —
  appropriate here, because the pool has exactly one entry point.
- **`from 10.255.0.4`** — the far-site router names 3560CG-2 as the advertising
  router. That attribution crosses the IPsec tunnel.
- **`forward metric 12`** — the internal cost to reach the ASBR, tracked
  separately from the external metric. This is exactly what a Type-1 external
  would have added and a Type-2 does not.

---

## 4. The IPsec tunnel is up — and both ends agree on the SAs

**Claim:** a route-based IKEv2 IPsec tunnel joins the two sites, terminating on
`st0.0` (Junos) and `tunnel.1` (PAN-OS).

**SRX345-LAB:**

```
Admin@SRX345-LAB> show security ike security-associations
Index   State  Initiator cookie  Responder cookie  Mode           Remote Address
2426501 UP     70d27ddcf8673513  1afe779f0f6c0e7c  IKEv2          [edge-west]

Admin@SRX345-LAB> show security ipsec security-associations
  Total active tunnels: 1     Total Ipsec sas: 1
  ID    Algorithm       SPI      Life:sec/kb  Mon lsys Port  Gateway
  <131073 ESP:aes-gcm-256/None 40e0e864 1533/ unlim - root 500 [edge-west]
  >131073 ESP:aes-gcm-256/None fde4bd0a 1533/ unlim - root 500 [edge-west]

Admin@SRX345-LAB> show interfaces terse st0.0
Interface               Admin Link Proto    Local                 Remote
st0.0                   up    up   inet     10.255.255.2/30
```

**PA440-LAB:**

```
admin@PA440-LAB> show vpn ike-sa
IKEv2 SAs
Gateway ID  Peer-Address  Gateway Name  Role  Algorithm             Established      ST
1           [edge-east]   GW-SRX        Init  PSK/DH14/A256/SHA256  Sep.13 02:09:08  Established

IKEv2 IPSec Child SAs
Gateway Name  TnID  Tunnel          Parent  Role  SPI(in)   SPI(out)  ST
GW-SRX        1     TUN-SRX:PROXY   7       Resp  FDE4BD0A  40E0E864  Mature

admin@PA440-LAB> show vpn ipsec-sa
GwID  TnID  Peer-Address  Tunnel(Gateway)         Algorithm  SPI(in)   SPI(out)  life(Sec/KB)    remain-time(Sec)
1     1     [edge-east]   TUN-SRX:PROXY(GW-SRX)   ESP/G256/  FDE4BD0A  40E0E864  3600/Unlimited  1827
```

### What to notice — the SPIs match across two vendors

| | SRX345 (Junos) | PA-440 (PAN-OS) |
|---|---|---|
| Inbound SPI | `40e0e864` | `FDE4BD0A` |
| Outbound SPI | `fde4bd0a` | `40E0E864` |

**The SRX's inbound SA is the PA's outbound SA, and vice versa.** A Security
Parameter Index identifies one specific unidirectional SA, so two devices naming
the same pair of values are describing the *same* tunnel — not two devices each
independently reporting success. Either box alone proves only that it thinks the
tunnel is up. Together they prove it is.

Also worth reading:

- **`Role Init` / `Role Resp`.** The PA initiated, the SRX responded. IKEv2 is
  not symmetric, and knowing which side initiates matters when debugging a
  tunnel that will not establish.
- **`PSK/DH14/A256/SHA256`** — pre-shared key, Diffie-Hellman group 14, AES-256,
  SHA-256.
- **`ESP:aes-gcm-256/None`.** The `/None` is not a missing algorithm. GCM is an
  AEAD cipher providing encryption *and* authentication in one operation, so
  there is no separate integrity algorithm to configure.
- **`remain-time(Sec) 1827`** of a `3600` lifetime — the SA is mid-life and will
  rekey. SPI values rotate on every rekey, which is why publishing them is
  harmless: they are ephemeral indices, not secrets.
- **`st0.0 ... 10.255.255.2/30`** — the tunnel is a routed interface with its own
  addressing, which is what allows OSPF to form an adjacency over it. A
  policy-based tunnel could not do this.

---

## 5. Link aggregation matched to what each device supports

**Claim:** uplinks are aggregated using LACP where the hardware supports it and
static EtherChannel where it does not — each choice matched to the device.

**3560CG-1 (EAST) — LACP on both bundles:**

```
LAB-CISCO-3560CG-1#show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(RU)         LACP      Gi0/1(P)    Gi0/2(P)
2      Po2(SU)         LACP      Gi0/7(P)    Gi0/8(P)
```

**3560CG-2 (WEST) — LACP up to the firewall, static down to the 2940:**

```
LAB-CISCO-3560CG-2#show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(RU)         LACP      Gi0/2(P)    Gi0/3(P)
2      Po2(SU)          -        Gi0/7(P)    Gi0/8(P)
```

**C2940-LAB (2003) — the other end of that static bundle:**

```
C2940-LAB#show etherchannel summary
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)          -        Fa0/7(Pd)   Fa0/8(P)
```

**SRX345-LAB — the Junos side of the EAST bundle:**

```
Admin@SRX345-LAB> show lacp interfaces ae0
Aggregated interface: ae0
    LACP state:       Role   Exp   Def  Dist  Col  Syn  Aggr  Timeout  Activity
      ge-0/0/6       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/6     Partner    No    No   Yes  Yes  Yes   Yes     Slow    Active
      ge-0/0/7       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/7     Partner    No    No   Yes  Yes  Yes   Yes     Slow    Active
```

**ARISTA710P-LAB:**

```
ARISTA710P-LAB>show port-channel
Port Channel Port-Channel2:
  Active Ports: Ethernet9 Ethernet10
```

### What to notice

- **The `Protocol` column is the whole story.** `LACP` on three bundles, **`-`
  on two** — 3560CG-2's `Po2` and the C2940's `Po1`, the two ends of the same
  link. That dash is the Catalyst 2940's IOS image having no LACP support, so
  the bundle is configured `mode on` at both ends.
- **A static bundle has no negotiation and therefore no safety net.** LACP
  verifies the far end agrees before forwarding; `mode on` simply assumes it, so
  a miscabled static EtherChannel is a layer-2 loop.
- **`Po1(RU)` vs `Po2(SU)`.** `R` = layer 3, `S` = layer 2, `U` = in use. The
  uplinks to the firewalls are routed port-channels carrying a `/30` — which is
  what lets OSPF run over them. The downlinks are switched trunks.
- **`Fast` actor / `Slow` partner on the SRX.** The timeout bit states what the
  sender wants to *receive*. The SRX asks for fast, so the Cisco transmits every
  1 s and the SRX can declare a member dead in 3 s; the Cisco asks for slow, so
  the SRX transmits every 30 s and the Cisco needs 90 s. Asymmetric detection
  times on one bundle — legal, and worth knowing before debugging a failover.

---

## 6. One time hierarchy across the fabric

**Claim:** the network devices synchronise to one internal authority, which
itself follows several external sources. **Five of the six do. The sixth is
below, with why.**

```
External NTP — 4 peers, all leap-second stepping
               (time.nist.gov + 0/1/2.pool.ntp.org)
        │
        ▼
CA-DC-01  10.20.2.5  — PDC emulator, the one internal authority
        │
        ├──► domain members   (automatic, via the domain hierarchy)
        └──► network devices  (explicit, configured per vendor)
```

**3560CG-1:**

```
LAB-CISCO-3560CG-1#show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.20.2.5       132.163.97.6     2      9     64   377  1.911  79.158  1.382
 * sys.peer, # selected, + candidate, - outlyer, x falseticker, ~ configured

LAB-CISCO-3560CG-1#show ntp status
Clock is synchronized, stratum 3, reference is 10.20.2.5
```

**3560CG-2:**

```
LAB-CISCO-3560CG-2#show ntp associations
  address         ref clock       st   when   poll reach  delay  offset   disp
*~10.20.2.5       132.163.97.6     2     26     64   377  1.066  64.141  2.386
```

**SRX345-LAB** — configured minutes before capture, four of eight polls in:

```
Admin@SRX345-LAB> show ntp associations
   remote         refid           st t when poll reach   delay   offset  jitter
===============================================================================
*10.20.2.5        132.163.97.6     2 -    3   64   17    1.574    1.699   3.180
```

**ARISTA710P-LAB:**

```
ARISTA710P-LAB>show ntp status
synchronised to NTP server (10.20.2.5) at stratum 3
   time correct to within 164 ms
   polling server every 64 s
```

**C2940-LAB** — the 2003 switch, captured later than the others:

```
C2940-LAB#show ntp associations

      address         ref clock     st  when  poll reach  delay  offset    disp
*~10.20.2.5        45.33.53.84       3    45    64  377     2.0    6.28     4.3
 * master (synced), # master (unsynced), + selected, - candidate, ~ configured
```

### The one that doesn't work: PA440-LAB

```
admin@PA440-LAB> show ntp

NTP state:
    NTP not synched, using local clock
    NTP server: 10.20.2.5
        status: error
        reachable: no
```

Configured correctly, and `reachable: no`. Every other device on the same
management plane reaches `10.20.2.5` without trouble, and the firewall itself is
reachable in-band on `loopback.1` — this document was written over SSH to it.

**The cause is a PAN-OS default, not a routing failure.** PAN-OS sources
management-plane services — NTP, DNS, updates, syslog — from the dedicated
**MGT interface**, independently of the dataplane routing table. The MGT port on
this unit is uncabled, so the query never leaves the box. Being reachable *on* a
loopback does not make the firewall send service traffic *from* it.

The fix is a **service route**: Device → Setup → Services → Service Route
Configuration → NTP → source it from `loopback.1`. Not yet applied, so the
firewall is running on its local clock and is the one device whose timestamps
should not be trusted for correlation.

### What to notice

- **`*` means selected, not merely configured.** `~` is "configured"; the
  asterisk marks the peer the system actually chose. A configured peer that
  never earns the asterisk is not being used.
- **`reach 377`** is octal — eight bits set, meaning the last eight polls all
  succeeded. `reach 0` on a freshly configured peer is normal; the poll interval
  is 64 seconds, so it takes several minutes to fill.
- **The `ref clock` column shows the DC's own upstream**, so each switch sees two
  levels up the chain rather than just its immediate parent.
- **That upstream changed between captures** — `132.163.97.6` (`time.nist.gov`,
  stratum 1, putting the DC at stratum 2) on the switches captured first,
  `45.33.53.84` (a pool server, leaving the DC at stratum 3) on the C2940
  captured later. Nothing was reconfigured in between. NTP's selection algorithm
  weighs delay, dispersion and jitter alongside stratum, and it re-selected on
  its own. A lower stratum is not automatically the chosen peer.
- **`x - falseticker` in the legend.** NTP's selection algorithm identifies and
  excludes a server that is confidently wrong, which only works with several
  sources to compare — so the DC runs four peers rather than one.
- **Every device is set to UTC, deliberately.** Local time is not monotonic:
  DST duplicates one hour every autumn and skips one every spring, so local
  timestamps are neither unique nor ordered across those boundaries. Correlating
  an event across six devices needs a clock that never repeats itself.

---

## 7. What is deliberately *not* here

- **WAN and edge addressing** is replaced with `[edge-west]` / `[edge-east]`,
  matching the topology diagram, which also omits it.
- **No credentials, keys, or pre-shared keys** appear in any output above. The
  IKE cookies and IPsec SPIs are protocol-level identifiers, not secrets, and
  they rotate on rekey.
- **No `show running-config` output.** Device configurations are backed up
  automatically to a self-hosted Git server and reviewed before anything is
  published — a running config contains SNMP communities, password hashes, and
  pre-shared keys, and no automated pipeline publishes one to a public
  repository.

---

*Captured 2026-09-13. This document is refreshed when the fabric changes.*
