# Configuration Guide

Annotated configuration for the key devices, so you can understand *why* each command is there and not just what it is. Interface numbers reflect the Packet Tracer build. Verify against `show running-config` on each device.

**Addressing recap:** internal subnets live in `132.11.0.0/21`, guest Wi-Fi is `132.11.6.0/24` (VLAN 80), and all point-to-point links use /30s from `132.11.8.0`.

---

## 1. Baseline hardening (applied to routers)

```
hostname <DEVICE-NAME>
enable secret <password>          ! privileged-mode password, stored hashed
service password-encryption       ! obscures any plaintext passwords in the config
banner motd # Unauthorized access is prohibited #
line console 0
 password <password>
 login
line vty 0 4
 transport input ssh              ! SSH only - no telnet, nothing in plaintext
 login local
ip domain-name smithlogistics.local
crypto key generate rsa           ! 1024+ bits, required for SSH
username admin secret <password>
```

Management traffic is a prime target. Telnet sends credentials in plaintext, SSH encrypts the whole session. The banner supports prosecution under the Computer Misuse Act 1990 by proving users were warned.

---

## 2. Core routers (R1, R2, R3)

Each core router has /30 links to both distribution switches and to a firewall. Example: R1.

```
interface GigabitEthernet0/0
 description Link to D1
 ip address 132.11.8.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 description Link to D2
 ip address 132.11.8.5 255.255.255.252
 no shutdown
interface GigabitEthernet0/2
 description Link to Firewall 1
 ip address 132.11.8.26 255.255.255.252
 no shutdown

router ospf 1
 network 132.11.8.0 0.0.0.3 area 0
 network 132.11.8.4 0.0.0.3 area 0
 network 132.11.8.24 0.0.0.3 area 0

ip route 0.0.0.0 0.0.0.0 132.11.8.25   ! default route toward the firewall/ISP
```

R2 and R3 follow the same pattern with their own /30s (R2: 132.11.8.8/30 to D1, 132.11.8.12/30 to D2, 132.11.8.28/30 to Firewall; R3: 132.11.8.16/30 to D1, 132.11.8.20/30 to D2, 132.11.8.32/30 to Firewall 2).

Why OSPF: with three core routers and six core-to-distribution links, static routes would need manual updates on every failure. OSPF re-converges in seconds. A core router can die and traffic reroutes automatically.

Why the static default: OSPF handles internal reachability, but "everything else" (the internet) is a single decision then sends it to the firewall.

---

## 3. Distribution switch D1 (VLANs 10–40)

Serves Warehouse, Transport, Admin, Customer Service.

```
ip routing                        ! turns the 3560 into a Layer 3 switch

vlan 10
 name Warehouse
vlan 20
 name Transport
vlan 30
 name Admin
vlan 40
 name CustService

interface Vlan10
 ip address 132.11.0.1 255.255.254.0    ! /23 - 510 hosts for 300 machines
interface Vlan20
 ip address 132.11.2.1 255.255.254.0    ! /23
interface Vlan30
 ip address 132.11.4.1 255.255.255.0    ! /24
interface Vlan40
 ip address 132.11.5.1 255.255.255.0    ! /24

! Uplinks to the core (verified in show ip ospf neighbor - all FULL)
interface GigabitEthernet0/1
 no switchport
 ip address 132.11.8.2 255.255.255.252   ! to R1
interface GigabitEthernet0/2
 no switchport
 ip address 132.11.8.10 255.255.255.252  ! to R2
interface FastEthernet0/24
 no switchport
 ip address 132.11.8.18 255.255.255.252  ! to R3

! Trunks down to access switches
interface range FastEthernet0/1 - 8
 switchport trunk encapsulation dot1q    ! 3560 requires this before trunk mode
 switchport mode trunk

router ospf 1
 network 132.11.0.0 0.0.1.255 area 0
 network 132.11.2.0 0.0.1.255 area 0
 network 132.11.4.0 0.0.0.255 area 0
 network 132.11.5.0 0.0.0.255 area 0
 network 132.11.8.0 0.0.0.3 area 0
 network 132.11.8.8 0.0.0.3 area 0
 network 132.11.8.16 0.0.0.3 area 0
```

Why `no switchport` on uplinks: it converts a switch port into a routed port with its own IP. This is what makes the /30 point-to-point links to the core work.

Why 802.1Q trunks: one physical cable to each access switch carries traffic for multiple VLANs, each frame tagged with its VLAN ID.

---

## 4. Distribution switch D2 (VLANs 50–80) including the guest ACLs

Serves IT, Management, Servers, and Guest Wi-Fi. Same structure as D1 (its own VLANs, /30 uplinks to R1/R2/R3, OSPF), plus the security centrepiece:

```
interface Vlan50
 ip address 132.11.7.1 255.255.255.128    ! IT /25
interface Vlan60
 ip address 132.11.7.129 255.255.255.192  ! Management /26
interface Vlan70
 ip address 132.11.7.193 255.255.255.224  ! Servers /27 (DMZ role)
interface Vlan80
 ip address 132.11.6.1 255.255.255.0      ! Guest /24
```

### Guest Wi-Fi isolation (the NACLs)

```
ip access-list extended BLOCK-GUEST
 permit ip 132.11.6.0 0.0.0.255 <ISP-range> <wildcard>   ! guests may reach the internet
 deny   ip 132.11.6.0 0.0.0.255 132.11.0.0 0.0.7.255     ! ...but no internal subnet
 permit ip any any

ip access-list extended BLOCK-TO-GUEST
 deny   ip 132.11.0.0 0.0.7.255 132.11.6.0 0.0.0.255     ! internal traffic cannot reach guests
 permit ip any any

interface Vlan80
 ip access-group BLOCK-GUEST in       ! filters traffic FROM guests
 ip access-group BLOCK-TO-GUEST out   ! filters traffic TO guests
```

Three things that matter here:

1. **Placement.** These ACLs must live on D2. The switch that owns VLAN 80's gateway. Applied anywhere else, guest traffic never crosses them and they filter nothing. A control outside the traffic path is no control at all.
2. **Direction.** `in` and `out` are from the perspective of the VLAN interface. Two ACLs in opposite directions create full two-way isolation.
3. **Verification.** `show access-lists` shows match counters next to each line once traffic hits them. Zero matches everywhere = the ACL exists but is not applied.

---

## 5. Access switches (2960, one per department)

Example: Warehouse switch, all ports in VLAN 10.

```
vlan 10
 name Warehouse

interface range FastEthernet0/1 - 20
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 2          ! at most 2 MAC addresses per port
 switchport port-security violation shutdown ! kill the port on violation

! Uplinks to D1 and D2 (dual-homed for redundancy)
interface GigabitEthernet0/1
 switchport mode trunk
interface GigabitEthernet0/2
 switchport mode trunk

! Every unused port: shut down
interface range FastEthernet0/21 - 24
 shutdown
```

Why port security: stops MAC flooding attacks and unauthorized devices. Why shutdown on unused ports: an open wall port is a free invitation onto the network.

STP note: with every access switch dual-homed to D1 and D2, Spanning Tree (PVST+) blocks one uplink per VLAN to prevent loops, and activates it automatically if the primary fails.

---

## 6. Verification commands

| Command | What it proves |
|---|---|
| `show ip interface brief` | Interfaces up, gateways assigned |
| `show ip ospf neighbor` | Adjacencies FULL — routing has converged |
| `show access-lists` | ACLs exist; match counters show they're filtering |
| `show ip interface vlan 80` | ACLs actually applied (in/out) on the guest gateway |
| `show cdp neighbors` | Physical cabling matches the design |
| `show port-security` | Port security active on access ports |

## 7. Saving
      Always save your work...
```
copy running-config startup-config
```

...on every device you change — **and then save the .pkt file itself**. The first survives a device reload while the second survives closing Packet Tracer. Skipping either silently loses your work.
