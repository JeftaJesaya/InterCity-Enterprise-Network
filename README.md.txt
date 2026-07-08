# InterCity Enterprise Network Implementation

## Project Overview

For this project, I designed and implemented a secure, scalable, and highly available enterprise network connecting three company branches:

* **Windhoek Headquarters**
* **Walvis Bay Branch**
* **Oshakati Branch**

I built the network using **Cisco Packet Tracer** to simulate a real-world enterprise environment. My objective was to create a secure multi-site enterprise network supporting departmental segmentation, centralized network services, dynamic routing, encrypted branch communication, and secure network administration.

---

## Network Architecture

The enterprise network I built consists of three geographical locations connected through redundant WAN connections.

### Branches

| Branch      | Purpose                                |
| ----------- | --------------------------------------- |
| Windhoek HQ | Main headquarters and central services |
| Walvis Bay  | Remote branch office                   |
| Oshakati    | Remote branch office                   |

I split each branch network into multiple departments using VLAN technology to improve security, performance, and network management.

---

## Network Technologies I Implemented

### VLAN Segmentation

I segmented each branch network into different VLANs based on department requirements.

| VLAN    | Department | Purpose                                      |
| ------- | ---------- | --------------------------------------------- |
| VLAN 10 | Management | Network administration and device management |
| VLAN 20 | Users      | Employee workstations                         |
| VLAN 30 | Servers    | Internal server resources                     |
| VLAN 40 | Guest      | Guest wireless and internet access            |
| VLAN 50 | VoIP       | Voice communication services                  |

This segmentation improves security by separating different departments and reducing unnecessary broadcast traffic.

---

## IP Addressing Design

I used private IPv4 addressing with the base network:

```text
10.0.0.0/16
```

I assigned each branch a separate address range:

| Branch      | Network Range |
| ----------- | ------------- |
| Windhoek HQ | 10.0.0.0/24   |
| Walvis Bay  | 10.0.1.0/24   |
| Oshakati    | 10.0.2.0/24   |

WAN point-to-point links use /30 subnetting.

Example:

```text
Windhoek ISP1:
10.0.3.0/30

Walvis Bay ISP1:
10.0.3.4/30

Oshakati ISP1:
10.0.3.8/30
```

---

## Routing Implementation

### OSPF Dynamic Routing

I used **Open Shortest Path First (OSPF)** as the interior gateway protocol.

I implemented OSPF to:

* Automatically exchange routing information between branches
* Provide scalability
* Support WAN redundancy
* Automatically select alternative paths during failures

All branch routers participate in OSPF Area 0.

Example:

```cisco
router ospf 10
 network 10.0.0.0 0.0.255.255 area 0
```

---

## Inter-VLAN Routing

I implemented inter-VLAN routing using the **router-on-a-stick** design.

I created router subinterfaces for each VLAN.

Example:

```cisco
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.0.0.65 255.255.255.240
```

This allows communication between required VLANs while maintaining logical network separation.

---

## Centralized DHCP Implementation

I set up a **centralized DHCP server** at Windhoek Headquarters to automatically assign IP addresses to devices across all branches.

I chose a centralized DHCP design to:

* Simplify IP address management
* Maintain consistent addressing policies
* Reduce configuration overhead
* Allow centralized administration of DHCP scopes

The DHCP server provides addressing services for:

* Windhoek HQ
* Walvis Bay Branch
* Oshakati Branch

### DHCP Relay Agent Configuration

Since DHCP requests are broadcast messages and routers don't forward broadcasts between networks, I configured **DHCP Relay Agents** on the router VLAN subinterfaces.

I used the `ip helper-address` command to forward DHCP requests from branch VLANs to the centralized DHCP server.

Example:

```cisco
interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.0.0.1 255.255.255.224
 ip helper-address 10.0.0.82
```

This lets devices in different VLANs and remote branches obtain IP addresses from the central DHCP server.

### DHCP Address Allocation

The DHCP server dynamically assigns:

* IP addresses
* Subnet masks
* Default gateways
* DNS information

to client devices.

Example DHCP networks:

| Branch     | VLAN          | Network      |
| ---------- | ------------- | ------------ |
| Windhoek   | Users VLAN 20 | 10.0.0.0/27  |
| Windhoek   | Guest VLAN 40 | 10.0.0.32/27 |
| Walvis Bay | Users VLAN 20 | 10.0.1.0/27  |
| Walvis Bay | Guest VLAN 40 | 10.0.1.32/27 |
| Oshakati   | Users VLAN 20 | 10.0.2.0/27  |
| Oshakati   | Guest VLAN 40 | 10.0.2.32/27 |

### Static IP Address Assignment for Printers

I configured printers with **static IP addresses** instead of DHCP, since they are fixed network resources that require predictable addressing.

Advantages include:

* Reliable access for users
* Easier troubleshooting
* Consistent printer identification
* Improved network management

---

## Network Security Implementation

I used Cisco Access Control Lists (ACLs) to control traffic flow between departments, protect network devices, and define encrypted VPN traffic.

My security design follows the principle of **least privilege**, where users only receive access required for their role.

### Inter-VLAN Security Using Extended ACLs

I implemented Extended ACL 100 on the **Users VLAN (VLAN 20)** and **Guest VLAN (VLAN 40)** router subinterfaces on all branch routers.

The purpose of ACL 100 is to enforce communication restrictions between different departments across the enterprise network.

The ACL implements the following security policies:

#### Guest VLAN Protection

I prevented guest networks from accessing any Management VLAN, protecting network infrastructure devices such as routers and switches from unauthorized access by guest users.

Blocked communication:

```text
Guest VLAN
      |
      X
Management VLAN
```

Guest networks:

```text
Windhoek Guest:
10.0.0.32/27

Walvis Bay Guest:
10.0.1.32/27

Oshakati Guest:
10.0.2.32/27
```

Management networks:

```text
Windhoek Management:
10.0.0.64/28

Walvis Bay Management:
10.0.1.64/28

Oshakati Management:
10.0.2.64/28
```

Example ACL rules:

```cisco
access-list 100 deny ip 10.0.0.32 0.0.0.31 10.0.0.64 0.0.0.15
access-list 100 deny ip 10.0.1.32 0.0.0.31 10.0.1.64 0.0.0.15
access-list 100 deny ip 10.0.2.32 0.0.0.31 10.0.2.64 0.0.0.15
```

#### User and Guest Network Isolation

I prevented User VLANs from communicating with Guest VLANs, so guest devices can't access internal user resources and employee devices can't directly communicate with guest networks.

Blocked communication:

```text
User VLAN
     |
     X
Guest VLAN
```

User networks:

```text
Windhoek Users:
10.0.0.0/27

Walvis Bay Users:
10.0.1.0/27

Oshakati Users:
10.0.2.0/27
```

Guest networks:

```text
Windhoek Guest:
10.0.0.32/27

Walvis Bay Guest:
10.0.1.32/27

Oshakati Guest:
10.0.2.32/27
```

Example ACL rule:

```cisco
access-list 100 deny ip 10.0.0.0 0.0.0.31 10.0.0.32 0.0.0.31
```

#### Allowing Required Communication

After applying these restrictions, I permitted all other legitimate traffic:

```cisco
access-list 100 permit ip any any
```

I applied the ACL inbound on the User and Guest VLAN subinterfaces:

```cisco
interface FastEthernet0/0.20
 ip access-group 100 in

interface FastEthernet0/0.40
 ip access-group 100 in
```

This ensures traffic is inspected before being routed to other networks.

---

### Secure Remote Management Using SSH

I secured remote administration using **SSH Version 2**.

I avoided Telnet because it transmits credentials in plaintext. SSH provides encrypted communication between administrators and network devices.

I configured SSH on:

* Branch routers
* Access switches

SSH configuration:

```cisco
ip ssh version 2
```

I configured local authentication using:

```cisco
username admin secret
```

### SSH Access Control Using Standard ACLs

I created Standard ACL 10 to restrict SSH access to only authorized Management VLAN networks, so only administrators located within the Management VLANs can remotely access network infrastructure devices.

Allowed management networks:

```text
Windhoek Management:
10.0.0.64/28

Walvis Bay Management:
10.0.1.64/28

Oshakati Management:
10.0.2.64/28
```

ACL configuration:

```cisco
access-list 10 permit 10.0.0.64 0.0.0.15
access-list 10 permit 10.0.1.64 0.0.0.15
access-list 10 permit 10.0.2.64 0.0.0.15
access-list 10 deny any
```

I applied the ACL to the VTY lines:

```cisco
line vty 0 4
 access-class 10 in
 login local
 transport input ssh
```

This gives me centralized secure administration across all branches.

Allowed:

```text
Management PC
10.0.0.67

        |
        |
       SSH

        |
        v

Routers and Switches
```

Blocked:

```text
Users VLAN
10.0.0.0/27

        X

SSH Access


Guest VLAN
10.0.0.32/27

        X

SSH Access
```

I applied the same SSH access policy to branch routers and switches.

---

### Site-to-Site VPN Implementation

I secured communication between branches using **IPsec site-to-site VPN tunnels**, providing confidentiality, integrity, and authentication for traffic travelling across WAN links.

I implemented the following IPsec security features:

* IKE Phase 1 negotiation
* AES encryption
* Pre-shared key authentication
* Diffie-Hellman Group 2
* IPsec ESP AES encryption
* SHA-HMAC authentication
* Perfect Forward Secrecy (PFS)

Example:

```cisco
crypto isakmp policy 10
 encr aes
 authentication pre-share
 group 2
```

### IPsec Crypto ACLs (Interesting Traffic)

I configured Crypto ACLs to define which traffic should be encrypted through the VPN tunnels. Only internal branch-to-branch communication is encrypted — traffic that doesn't match the crypto ACL rules is transmitted normally.

#### Windhoek to Walvis Bay VPN Tunnel

ACL 110 defines encrypted traffic between Windhoek and Walvis Bay networks.

```cisco
access-list 110 permit ip 10.0.0.0 0.0.0.255 10.0.1.0 0.0.0.255
```

Encrypted traffic:

```text
Windhoek LAN
      |
      |
   IPsec VPN Tunnel
      |
      |
Walvis Bay LAN
```

#### Windhoek to Oshakati VPN Tunnel

ACL 111 defines encrypted traffic between Windhoek and Oshakati networks.

```cisco
access-list 111 permit ip 10.0.0.0 0.0.0.255 10.0.2.0 0.0.0.255
```

Encrypted traffic:

```text
Windhoek LAN
      |
      |
   IPsec VPN Tunnel
      |
      |
Oshakati LAN
```

I referenced the crypto ACLs in the crypto maps:

```cisco
crypto map IPSEC-MAP-ISP1 10 ipsec-isakmp
 set peer 10.0.3.5
 set transform-set ENTERPRISE-IPSEC
 match address 110
```

This ensures only traffic matching the defined ACL entries is encrypted.

---

## High Availability Design

I designed the network with redundancy and availability in mind.

Features I implemented:

* Dual ISP WAN connections
* Primary and backup WAN links
* OSPF cost-based path selection
* Redundant IPsec VPN tunnels

The primary ISP links use lower OSPF costs, while backup links have higher costs. If a primary WAN connection fails, OSPF automatically selects the backup path.

---

## Wireless Network Integration

I integrated wireless access points into the enterprise LAN. Wireless users are assigned to appropriate VLANs to maintain security and segmentation, and guest wireless traffic is kept separate from internal company resources through VLAN isolation.

---

## Skills Demonstrated

* Enterprise network design
* IPv4 addressing and subnetting
* Centralized DHCP deployment
* DHCP relay configuration
* Static IP assignment
* VLAN implementation
* Router-on-a-stick configuration
* OSPF dynamic routing
* ACL security implementation
* SSH secure administration
* IPsec VPN configuration
* WAN design
* Network troubleshooting
* Cisco IOS administration

---

## Project Outcome

Through this project, I built a realistic multi-branch enterprise network capable of supporting business operations while maintaining:

✅ Centralized IP address management
✅ Secure departmental segmentation
✅ Controlled access between VLANs
✅ Secure remote administration
✅ Dynamic routing between branches
✅ Encrypted site-to-site communication
✅ WAN redundancy
✅ Scalable enterprise architecture

This project provided me with practical experience in designing, configuring, securing, and troubleshooting enterprise Cisco network infrastructure.