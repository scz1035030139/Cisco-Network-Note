# Cisco IOS-XE DMVPN 与 FlexVPN 技术总结

> 本文汇总 DMVPN、FlexVPN、NHRP、IKEv2、动态 VTI、spoke-to-spoke 和 IKEv2 Redirect 的讨论。配置为示意片段，需根据 IOS-XE 版本核实。

## Part I — Architecture and Comparison

## Executive summary

DMVPN and FlexVPN are architectures for encrypted enterprise VPNs. DMVPN commonly combines multipoint GRE (mGRE), NHRP, and IPsec. FlexVPN is an IKEv2-based framework using virtual tunnel interfaces (VTI), including dynamic VTI (dVTI) built from Virtual-Template interfaces.

**Important:** FlexVPN does not inherently provide automatic direct spoke-to-spoke forwarding. Cisco's documented FlexVPN spoke-to-spoke shortcut feature uses NHRP Redirect/Shortcut alongside IKEv2 and dynamic VTI. IKEv2 Redirect is a separate gateway redirection feature and is not a drop-in NHRP replacement.

## Comparison

| Dimension | DMVPN | FlexVPN |
|---|---|---|
| Common tunnel model | mGRE shared multipoint tunnel | Static VTI and/or dynamic VTI (Virtual-Access) |
| VPN security | IPsec, with IKEv1 or IKEv2 depending on configuration | IPsec with IKEv2 |
| Peer reachability discovery | NHRP maps overlay addresses to NBMA addresses | IKEv2 establishes peer sessions; NHRP also used for documented spoke-to-spoke shortcut feature |
| Hub acceptance of unknown spokes | NHRP registration and IPsec peer policies | IKEv2 profile and Virtual-Template create per-peer Virtual-Access |
| Direct spoke-to-spoke | NHRP-based Phase 2/3 designs | NHRP Redirect/Shortcut + peer-specific IKEv2/IPsec dVTI in Cisco's shortcut design |
| Hub transit forwarding | Possible | Possible |
| Routing | BGP, EIGRP, OSPF, etc., with topology-specific constraints | BGP, EIGRP, OSPF, etc., with topology-specific constraints |

## Two address spaces

- Overlay/tunnel addresses: e.g., hub `10.0.0.1`, spoke A `10.0.0.11`, spoke B `10.0.0.12`.
- Underlay/NBMA addresses: e.g., hub `192.0.2.1`, spoke A `192.0.2.11`, spoke B `192.0.2.12`.

The underlay routes packets to WAN addresses. Overlay routing selects the VPN next hop. NHRP in DMVPN resolves the mapping needed for mGRE encapsulation.

## Hub-and-spoke versus shortcut

```text
Normal hub transit:
LAN-A -- Spoke-A ==ESP== Hub ==ESP== Spoke-B -- LAN-B

Direct shortcut:
LAN-A -- Spoke-A ==========ESP========== Spoke-B -- LAN-B
                         (Hub remains available)
```

In hub transit, the hub decrypts inbound IPsec, routes/policies the plaintext packet, then encrypts it into the other spoke's IPsec SA. A direct shortcut avoids that transit workload.

## Important correction to our discussion

An earlier simplified answer described IKEv2 Redirect as the mechanism for direct FlexVPN spoke-to-spoke. That was incorrect for Cisco's documented shortcut implementation. IKEv2 Redirect redirects a client toward another VPN gateway; NHRP Redirect/Shortcut discovers and triggers spoke-to-spoke shortcuts.

## References

- Cisco, *FlexVPN Spoke-to-Spoke*: https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-flex-spoke.html
- Cisco, *FlexVPN Spoke-to-Spoke (IOS-XE)*: https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-flex-spoke.html
- RFC 7296, *Internet Key Exchange Protocol Version 2*: https://www.rfc-editor.org/rfc/rfc7296
- RFC 5685, *Redirect Mechanism for IKEv2*: https://www.rfc-editor.org/rfc/rfc5685

---

## Part II — Control Plane and Packet Flows

## 1. DMVPN registration and resolution

```text
Spoke A                   Hub/NHS                  Spoke B
   |-- NHRP Registration -->|                         |
   |                        |<-- NHRP Registration ---|
   |                        |                         |
   |-- Data via Hub --------|------------------------>|
   |<-- NHRP Redirect ------|    (DMVPN Phase 3)     |
   |-- NHRP Resolution Request ---------------------->|
   |<--------------------- NHRP Resolution Reply -----|
   |========= Direct GRE protected by IPsec =========>|
```

This diagram is conceptual: NHRP resolution requests can traverse the NHS and the exact exchanges depend on DMVPN phase and routing. Phase 1 is hub-only, Phase 2 allows direct peer mappings, and Phase 3 uses redirect/shortcut signaling to optimize hub-routed traffic.

### Why NHRP?

An mGRE interface has no single configured `tunnel destination`. When a route resolves to overlay next hop `10.0.0.12`, the router needs its NBMA address `192.0.2.12` to form the outer GRE/IP header. NHRP provides the mapping.

```text
Original destination 10.20.20.10
   -> overlay route next hop 10.0.0.12
   -> NHRP mapping 10.0.0.12 = 192.0.2.12
   -> outer GRE/IP destination 192.0.2.12
   -> IPsec protection and underlay delivery
```

## 2. FlexVPN dynamic hub tunnel creation

```text
Spoke A                         Hub
  |----- IKE_SA_INIT ----------->|
  |<---- IKE_SA_INIT ------------|
  |----- IKE_AUTH -------------->|
  |<---- IKE_AUTH ---------------|
  |                              | Clone Virtual-Template
  |                              | -> Virtual-Access per peer
  |========= ESP / Child SA =====|
```

A spoke typically has a configured VTI to the hub. The hub can accept dynamically identified peers using an IKEv2 profile and clone its Virtual-Template into a Virtual-Access interface. IKEv2 creates/authenticates the IKE SA and negotiates Child SAs; ESP encrypts data.

## 3. FlexVPN spoke-to-spoke shortcut

```text
Spoke A                   Hub                     Spoke B
  |---- Data via hub ----->|----------------------->|
  |<--- NHRP Redirect ----|                        |
  |---- NHRP resolution (via NHRP infrastructure) ->|
  |<--- Resolution reply / peer information -------|
  |                                                 |
  |-------- IKE_SA_INIT --------------------------->|
  |<------- IKE_SA_INIT ----------------------------|
  |-------- IKE_AUTH ------------------------------>|
  |<------- IKE_AUTH -------------------------------|
  |    Dynamic peer-specific Virtual-Access / IPsec |
  |================ Direct ESP ====================>|
```

**Caveat:** This is a logical sequence, not a literal packet capture; detailed NHRP and IKEv2 signaling depends on IOS-XE release, configuration, NAT and topology. The routing/shortcut state must also steer the relevant traffic to the direct peer.

## 4. NHRP Redirect versus IKEv2 Redirect

| Question | NHRP Redirect | IKEv2 Redirect |
|---|---|---|
| What does it signal? | A more direct overlay forwarding path may exist | Client should use another VPN gateway |
| Used for Cisco FlexVPN spoke shortcuts? | Yes | Not as the documented replacement |
| Does it itself encrypt packets? | No | No |
| Does it negotiate a Child SA? | No | IKEv2 negotiations following redirect do |

## 5. Three forwarding decisions

1. **Overlay routing:** Which next hop or VPN tunnel should carry the destination prefix?
2. **Shortcut/peer resolution:** For direct paths, how is the correct remote spoke discovered (NHRP in Cisco's documented shortcut designs)?
3. **Underlay routing:** How does the physical network reach the remote WAN/NBMA address?

## 6. Operational verification

```cisco
show crypto ikev2 sa
show crypto ipsec sa
show crypto session
show ip nhrp
show ip route
show ip interface brief | include Tunnel|Virtual-Access
```

On DMVPN, additionally inspect `show dmvpn`. On FlexVPN, look for new Virtual-Access interfaces and peer-specific IKEv2/IPsec sessions when shortcuts form. Exact output and supported commands vary by IOS-XE version.

## References

- https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-flex-spoke.html
- https://www.rfc-editor.org/rfc/rfc2332
- https://www.rfc-editor.org/rfc/rfc7296
- https://www.rfc-editor.org/rfc/rfc5685

---

## Part III — Configuration Building Blocks

> **Scope:** These are illustrative building blocks, not a copy/paste production configuration. Complete implementations require compatible IOS-XE releases, interface/VRF choices, authentication and certificate/PSK handling, routing policy, NAT traversal, and tunnel addressing. Do not assume a local IP pool alone automatically negotiates spoke addresses; validate the relevant IKEv2 configuration/address-assignment mechanism for your release.

## 1. DMVPN: typical hub tunnel

```cisco
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 ip nhrp authentication DMVPN-AUTH
 ip nhrp network-id 100
 ip nhrp redirect
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile DMVPN-IPSEC
```

## 2. DMVPN: typical spoke tunnel

```cisco
interface Tunnel0
 ip address 10.0.0.11 255.255.255.0
 ip nhrp authentication DMVPN-AUTH
 ip nhrp network-id 100
 ip nhrp nhs 10.0.0.1
 ip nhrp map 10.0.0.1 192.0.2.1
 ip nhrp map multicast 192.0.2.1
 ip nhrp shortcut
 tunnel source GigabitEthernet1
 tunnel mode gre multipoint
 tunnel key 100
 tunnel protection ipsec profile DMVPN-IPSEC
```

These snippets omit the IPsec profile, IKE configuration, WAN interfaces, and routing. The `ip nhrp redirect`/`shortcut` pairing illustrates a Phase 3 design. WAN addresses shown are documentation examples.

## 3. FlexVPN: dynamic hub building blocks

```cisco
crypto ikev2 profile FLEX-IKEV2
 match identity remote any
 authentication remote pre-share
 authentication local pre-share
 keyring local FLEX-KEYRING
 virtual-template 1
!
crypto ipsec profile FLEX-IPSEC
 set transform-set FLEX-TS
 set ikev2-profile FLEX-IKEV2
!
interface Loopback0
 ip address 10.255.0.1 255.255.255.255
!
interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile FLEX-IPSEC
```

This shows the essential relationship between IKEv2 profile, IPsec profile, Virtual-Template and per-peer Virtual-Access creation. Define `FLEX-KEYRING` and `FLEX-TS` separately and verify IOS-XE release-specific syntax. For real deployments prefer peer-scoped identities and stronger authentication instead of an unrestricted wildcard profile.

## 4. FlexVPN: static spoke VTI building blocks

```cisco
interface Tunnel0
 description FlexVPN to hub
 ip address negotiated
 tunnel source GigabitEthernet1
 tunnel destination 192.0.2.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile FLEX-IPSEC
```

`ip address negotiated` requires an actual compatible address-assignment configuration on the hub. Alternatively, use a supported static tunnel-addressing design. The spoke's IKEv2 profile/keyring must authenticate the hub.

## 5. FlexVPN spoke-to-spoke: additional concepts

Cisco's documented spoke-to-spoke design uses NHRP redirect/shortcut features on the appropriate tunnel/Virtual-Template interfaces and a dynamic VTI capable of establishing direct peer sessions. Illustrative feature commands:

```cisco
! Hub-side NHRP shortcut signaling
interface Virtual-Template1 type tunnel
 ip nhrp network-id 1
 ip nhrp redirect
!
! Spoke's hub-facing tunnel
interface Tunnel0
 ip nhrp shortcut 1
!
! Spoke's shortcut template
interface Virtual-Template1 type tunnel
 ip nhrp network-id 1
 ip nhrp shortcut 1
 tunnel protection ipsec profile FLEX-IPSEC
```

**Not sufficient alone:** A working shortcut requires the rest of Cisco's documented NHRP/IKEv2 configuration, peer authentication, routing and appropriate interface associations. Follow the release-specific configuration guide rather than appending these commands blindly to a basic hub-and-spoke setup.

## 6. Routing and behavior decisions

- **Hub-only forwarding required:** Do not enable spoke-to-spoke shortcuts. Ensure the overlay routing and policy steer inter-spoke traffic through the hub.
- **Direct forwarding required:** Use a documented shortcut design and validate NHRP resolution, direct IKEv2 SAs, Virtual-Access creation, and resulting routes.
- **OSPF over dynamic tunnels:** Verify OSPF network type, interface inheritance from Virtual-Template, passive-interface behavior and adjacency formation on cloned Virtual-Access interfaces.
- **NAT:** Check NAT-T, WAN reachability and dynamic peer initiation behavior.

## 7. Useful commands

```cisco
show crypto ikev2 sa detail
show crypto session detail
show crypto ipsec sa
show ip nhrp
show dmvpn
show ip interface brief
show ip route
show ip ospf neighbor
```

## References

- Cisco IOS-XE FlexVPN Spoke-to-Spoke: https://www.cisco.com/c/en/us/td/docs/routers/ios-xe/security-vpn/security-vpn/m_sec-flex-spoke.html
- Cisco IOS 17.x FlexVPN Spoke-to-Spoke: https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec-flex-spoke.html
