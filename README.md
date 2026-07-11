# CCNA-Lab-ACL-Standard-Extended
CCNA lab demonstrating Standard vs. Extended Access Control Lists in Cisco Packet Tracer, host/service-level traffic filtering, static routing, and verified test results across R-1, R-2, and R-3.


## 1. Objective

This lab builds on the static-routing topology from prior labs and introduces **Access Control Lists (ACLs)** as a traffic-filtering mechanism. Two ACL types are implemented and tested independently:

- An **Extended ACL** on **R-1**, filtering specific hosts by protocol/port (Layer 4 granularity).
- A **Standard ACL** on **R-3**, filtering specific hosts by source address only (Layer 3 granularity).

The goal is to demonstrate how each ACL type restricts traffic differently, verify the restrictions through live testing, and confirm that unaffected traffic continues to flow normally end-to-end across all three routing domains.

---

## 2. Topology Overview

A 3-router, fully-routed topology connects three separate LANs over two serial WAN links:

| Router | LAN Interface | LAN Subnet | Connected Hosts |
|---|---|---|---|
| R-1 | Gig0/0/0 — 192.168.10.1/24 | 192.168.10.0/24 | PC0, PC1, PC2 |
| R-2 | Gig0/0/0 — 192.168.20.1/24 | 192.168.20.0/24 | PC3, PC4 |
| R-3 | Gig0/0/0 — 192.168.30.1/24 | 192.168.30.0/24 | Server0 (192.168.30.2) |

| WAN Link | Interfaces | Subnet |
|---|---|---|
| R-1 ↔ R-2 | R-1 Se0/1/0 (12.0.0.1) — R-2 Se0/1/0 (12.0.0.2) | 12.0.0.0/8 |
| R-2 ↔ R-3 | R-2 Se0/1/1 (23.0.0.1) — R-3 Se0/1/1 (23.0.0.2) | 23.0.0.0/8 |

R-2 acts as the central transit router between the two edge routers, R-1 and R-3, which is where each ACL is respectively applied.

---

## 3. Device Configuration Summary

### R-1
```
hostname R-1
interface gig0/0/0
 ip address 192.168.10.1 255.255.255.0
 no shutdown
interface serial0/1/0
 ip address 12.0.0.1 255.0.0.0
 no shutdown
```

### R-2
```
hostname R-2
interface gig0/0/0
 ip address 192.168.20.1 255.255.255.0
 no shutdown
interface serial0/1/0
 ip address 12.0.0.2 255.0.0.0
 no shutdown
interface serial0/1/1
 ip address 23.0.0.1 255.0.0.0
 no shutdown
```

### R-3
```
hostname R-3
interface gig0/0/0
 ip address 192.168.30.1 255.255.255.0
 no shutdown
interface serial0/1/1
 ip address 23.0.0.2 255.0.0.0
 no shutdown
```

---

## 4. Static Routing

Since this lab focuses on ACL behavior rather than dynamic routing, full reachability between all three LANs is established using static routes.

**R-1**
```
ip route 192.168.20.0 255.255.255.0 12.0.0.2
ip route 192.168.30.0 255.255.255.0 12.0.0.2
ip route 23.0.0.0 255.0.0.0 12.0.0.2
```

**R-2**
```
ip route 192.168.10.0 255.255.255.0 12.0.0.1
ip route 192.168.30.0 255.255.255.0 23.0.0.2
```

**R-2** does not need static routes for the 12.0.0.0/8 or 23.0.0.0/8 networks since both are directly connected — this is confirmed in its routing table.

**R-3**
```
ip route 192.168.20.0 255.255.255.0 23.0.0.1
ip route 192.168.10.0 255.255.255.0 23.0.0.1
ip route 12.0.0.0 255.0.0.0 23.0.0.1
```

`show ip route` on all three routers confirms full reachability: each router has connected routes to its directly-attached networks and static (S) routes to every remote subnet, with no gaps in the routing tables.

---

## 5. Extended ACL — Configured on R-1

Extended ACLs filter based on source, destination, protocol, and port — enabling granular, per-service control. ACL **110** is built to selectively restrict two specific hosts on R-1's LAN from reaching specific services on the Server (192.168.30.2), while leaving everything else untouched.

```
access-list 110 deny tcp host 192.168.10.2 host 192.168.30.2 eq www
access-list 110 deny tcp host 192.168.10.3 host 192.168.30.2 eq ftp
access-list 110 permit ip any any
!
interface gig0/0/0
 ip access-group 110 in
```

**Rule logic:**
- Line 1 blocks **192.168.10.2 (PC0)** from reaching the server over **HTTP** only.
- Line 2 blocks **192.168.10.3 (PC1)** from reaching the server over **FTP** only.
- Line 3 is a permit-all statement, explicitly allowing every other packet — this is required because the implicit `deny any any` at the end of every ACL would otherwise block all traffic once any `deny` line exists.
- The ACL is applied **inbound** on Gig0/0/0 — the interface facing the LAN — so it evaluates traffic as it enters the router from PC0/PC1/PC2, before it is routed anywhere else.

### Verification

| Test | Source | Result | Interpretation |
|---|---|---|---|
| HTTP to `http://192.168.30.2` | PC0 (192.168.10.2) | **Request Timeout** | Confirms ACL 110 line 1 is blocking HTTP specifically for this host |
| Ping + FTP login to 192.168.30.2 | Unrestricted host on R-1 LAN | Ping 0% loss; FTP session connects, authenticates (`saddam`), and logs in successfully | Confirms hosts *not* named in the ACL retain full, unrestricted access |
| FTP to 192.168.30.2 | PC1 (192.168.10.3) | **Timed out / connection failed** | Confirms ACL 110 line 2 is blocking FTP specifically for this host |

This set of results confirms the extended ACL is filtering **per-host, per-protocol** — PC0 loses only HTTP, PC1 loses only FTP, and any other host on the same LAN keeps full access to all services, including ICMP and FTP.

---

## 6. Standard ACL — Configured on R-3

Standard ACLs can only match on **source address**, making them less granular — a match blocks *all* traffic from that host, regardless of protocol or destination. ACL **10** is built to block two named hosts from R-1's LAN from reaching R-3's LAN entirely.

```
access-list 10 deny host 192.168.10.2
access-list 10 deny host 192.168.10.3
access-list 10 permit any
!
interface gig0/0/0
 ip access-group 10 out
```

**Rule logic:**
- Denies all traffic sourced from **192.168.10.2** and **192.168.10.3**.
- Permits everything else.
- Applied **outbound** on Gig0/0/0 — the interface facing the local LAN/server. This is a key placement decision for standard ACLs: because they can't match on destination, they are best applied as close to the destination as possible (out, near the server) to avoid accidentally blocking those hosts' traffic to *other* destinations elsewhere in the network.

### Verification

| Test | Source | Result | Interpretation |
|---|---|---|---|
| Ping to 192.168.30.2 | Unrestricted host on R-1 LAN | 0% loss, replies received | Confirms hosts not named in ACL 10 pass through cleanly |
| Ping to 192.168.30.2 | 192.168.10.2 (PC0) | Destination host unreachable | Confirms ACL 10 is dropping this host's traffic before it reaches the server subnet |
| Ping to 192.168.30.2 | 192.168.10.3 (PC1) | Destination host unreachable | Confirms ACL 10 is also dropping this second denied host |
| Ping to 192.168.30.2 | Host on R-2's LAN (192.168.20.0/24) | 0% loss, replies received (TTL=126) | Confirms the ACL only targets the two explicitly named hosts — traffic from an entirely different network is unaffected |

This confirms standard ACL 10 is filtering **strictly by source IP**, with no regard to which service is being requested — unlike the extended ACL on R-1, which permitted ICMP and other services for the same two hosts while blocking only HTTP/FTP respectively.

---

## 7. Key Concepts Demonstrated

- **Standard vs. Extended ACL granularity** — Standard ACLs (1–99) match source address only and produce an all-or-nothing block per host; Extended ACLs (100–199) can match source, destination, protocol, and port, enabling selective, per-service restrictions.
- **Placement best practice** — The extended ACL was applied close to the *source* (R-1's LAN-facing interface, inbound), while the standard ACL was applied close to the *destination* (R-3's LAN-facing interface, outbound) — following the standard Cisco design guideline for each ACL type.
- **Implicit deny any** — Both ACLs required an explicit `permit` statement at the end; without it, the implicit deny at the bottom of every ACL would have silently blocked all traffic once the first `deny` line was added.
- **Top-down, first-match processing** — Each ACL evaluates statements sequentially and stops at the first match, which is why host-specific `deny` entries were listed before the final `permit any`.
- **Directional filtering (in vs. out)** — `in` evaluates traffic as it enters an interface (before routing); `out` evaluates it just before it leaves an interface (after routing). The lab uses both directions across the two ACLs, reinforcing when each is appropriate.

---

## 8. Conclusion

Both ACL types were successfully configured, applied, and verified against their intended targets. The extended ACL on R-1 demonstrated precise, service-level filtering — blocking HTTP for one host and FTP for another while preserving all other connectivity. The standard ACL on R-3 demonstrated coarser, host-level filtering — fully blocking two named hosts from reaching the server's subnet while leaving all unrelated traffic, including an entirely separate LAN, unaffected. Testing at each stage confirmed the ACLs behaved exactly as configured, with no unintended side effects on routing or connectivity elsewhere in the topology.
