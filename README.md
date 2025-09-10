Two-Site Enterprise Network (Cisco Packet Tracer)

What this is: a Cisco Packet Tracer lab that simulates a small company with an HQ and a Branch connected over a WAN, plus a simulated ISP/Internet.
You’ll see VLANs, inter-VLAN routing at HQ, OSPF between sites, and NAT/PAT to reach an Internet test PC.

Open TwoSiteEnterprise.pkt in Packet Tracer (v8+). Start the devices, apply or verify the configs below, then run the test checklist.

Topology & Models:
ISP, R1 (HQ), R2 (Branch): Cisco 2911
SW-HQ-L3: Catalyst 3560 (Layer-3 switch)
SW-HQ-ACC, SW-BR-ACC: Catalyst 2960
End hosts: 4× PCs (PC1–PC4) + 1× Server (HQ), 1× Internet PC

IP Plan (Summary)
| Link / VLAN             | Network             | Interfaces                                             |
| ----------------------- | ------------------- | ------------------------------------------------------ |
| **ISP ↔ R1 WAN**        | **203.0.113.0/30**  | ISP `G0/0=203.0.113.1`, R1 `G0/0=203.0.113.2`          |
| **ISP ↔ Internet-PC**   | **203.0.113.8/30**  | ISP `G0/1=203.0.113.9`, Internet-PC `Fa0=203.0.113.10` |
| **R1 ↔ SW-HQ-L3**       | **10.10.254.0/30**  | R1 `G0/1=10.10.254.1`, SW-HQ-L3 `G0/1=10.10.254.2`     |
| **R1 ↔ R2 (Serial)**    | **10.255.255.0/30** | R1 `S0/0/0=10.255.255.1`, R2 `S0/0/0=10.255.255.2`     |
| **HQ VLAN10 Users**     | **10.10.10.0/24**   | Gateway `10.10.10.1` (SVI on SW-HQ-L3)                 |
| **HQ VLAN20 Servers**   | **10.10.20.0/24**   | Gateway `10.10.20.1` (SVI on SW-HQ-L3)                 |
| **HQ VLAN99 Mgmt**      | **10.10.99.0/24**   | Gateway `10.10.99.1` (SVI on SW-HQ-L3)                 |
| **Branch VLAN10 Users** | **10.20.10.0/24**   | Gateway `10.20.10.1` (R2 `G0/0`)                       |

PCs / Server (static examples):
Internet-PC: 203.0.113.10/30, GW 203.0.113.9, DNS 8.8.8.8
PC1: 10.10.10.21/24, GW 10.10.10.1
PC2: 10.10.10.22/24, GW 10.10.10.1
Server: 10.10.20.10/24, GW 10.10.20.1
PC3: 10.20.10.21/24, GW 10.20.10.1
PC4: 10.20.10.22/24, GW 10.20.10.1

Port Map (who plugs where)
ISP:
G0/0 ↔ R1 G0/0
G0/1 ↔ Internet-PC Fa0
R1 (HQ):
G0/0 ↔ ISP G0/0
G0/1 ↔ SW-HQ-L3 G0/1 (routed /30)
S0/0/0 ↔ R2 S0/0/0
SW-HQ-L3 (3560):
G0/1 routed ↔ R1 G0/1
G0/2 trunk ↔ SW-HQ-ACC G0/1
SW-HQ-ACC (2960):
G0/1 trunk ↔ SW-HQ-L3 G0/2
F0/1 → PC1 (VLAN10), F0/2 → PC2 (VLAN10), F0/3 → Server (VLAN20)
R2 (Branch):
S0/0/0 ↔ R1 S0/0/0
G0/0 L3 access 10.20.10.1/24 ↔ SW-BR-ACC G0/1
SW-BR-ACC (2960):
G0/1 access VLAN10 ↔ R2 G0/0
F0/1 → PC3 (VLAN10), F0/2 → PC4 (VLAN10)

Full Device Configs:
ISP (2911)
  conf t
hostname ISP
interface g0/0
 ip address 203.0.113.1 255.255.255.252
 no shut
interface g0/1
 ip address 203.0.113.9 255.255.255.252
 no shut
! route back to private space via R1
ip route 10.0.0.0 255.0.0.0 203.0.113.2
end
wr

R1 (HQ, 2911) — WAN, LAN, Serial, OSPF, NAT
  conf t
hostname R1
! WAN to ISP
interface g0/0
 ip address 203.0.113.2 255.255.255.252
 ip nat outside
 no shut
! Routed link to HQ L3 switch
interface g0/1
 ip address 10.10.254.1 255.255.255.252
 ip nat inside
 no shut
! Serial to Branch
interface s0/0/0
 ip address 10.255.255.1 255.255.255.252
 clock rate 64000
 ip nat inside
 no shut

! Default route to ISP
ip route 0.0.0.0 0.0.0.0 203.0.113.1

! OSPF area 0
router ospf 1
 router-id 2.2.2.2
 network 10.10.254.0 0.0.0.3 area 0
 network 10.255.255.0 0.0.0.3 area 0
 default-information originate always

! NAT/PAT for all private ranges
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface g0/0 overload
end
wr

SW-HQ-L3 (3560) — Gateways (SVIs) + Trunk Downlink + OSPF :
  conf t
hostname SW-HQ-L3
ip routing

! Routed /30 uplink to R1
interface g0/1
 no switchport
 ip address 10.10.254.2 255.255.255.252
 no shut

! Trunk down to the access switch
interface g0/2
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shut

! VLANs
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name MGMT

! SVIs (default gateways at HQ)
interface vlan 10
 ip address 10.10.10.1 255.255.255.0
 no shut
interface vlan 20
 ip address 10.10.20.1 255.255.255.0
 no shut
interface vlan 99
 ip address 10.10.99.1 255.255.255.0
 no shut

! OSPF: advertise HQ subnets
router ospf 1
 router-id 1.1.1.1
 network 10.10.254.0 0.0.0.3 area 0
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.99.0 0.0.0.255 area 0
 passive-interface default
 no passive-interface g0/1
 no passive-interface g0/2
end
wr

SW-HQ-ACC (2960) — Trunk Uplink + Access Ports :
conf t
hostname SW-HQ-ACC
vlan 10
 name USERS
vlan 20
 name SERVERS

interface g0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shut

interface f0/1
 switchport mode access
 switchport access vlan 10
 no shut
interface f0/2
 switchport mode access
 switchport access vlan 10
 no shut
interface f0/3
 switchport mode access
 switchport access vlan 20
 no shut
end
wr

R2 (Branch, 2911) — Serial, L3 Access to Users, OSPF :
conf t
hostname R2
interface s0/0/0
 ip address 10.255.255.2 255.255.255.252
 no shut

! Single-VLAN site (simple): L3 on G0/0
interface g0/0
 ip address 10.20.10.1 255.255.255.0
 no shut

router ospf 1
 router-id 3.3.3.3
 network 10.255.255.0 0.0.0.3 area 0
 network 10.20.10.0 0.0.0.255 area 0
end
wr

SW-BR-ACC (2960) — Access Ports for Users :
conf t
hostname SW-BR-ACC
vlan 10
 name USERS
interface g0/1
 switchport mode access
 switchport access vlan 10
 no shut
interface f0/1
 switchport mode access
 switchport access vlan 10
 no shut
interface f0/2
 switchport mode access
 switchport access vlan 10
 no shut
end
wr

Notes :
Use copper straight-through for router/switch/PC links; Serial DCE/DTE for the WAN.
If a link is amber/red: check you’re on the correct port, no shut, and the right mode (trunk vs access).
On the 3560, set trunk encapsulation dot1q before switchport mode trunk.

