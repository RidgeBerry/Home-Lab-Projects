# Hybrid Multi-Vendor BGP & High Availabity Lab

---

<aside>
📜

**TABLE OF CONTENTS**

</aside>

# **Introduction**

This lab served as my introduction to BGP and GNS3, and it stands as the most challenging infrastructure project I have completed to date. The architecture consisted of two simulated environments: a Virtual Service Provider (ISP) and a Physical Enterprise Edge.

My objective was to see and manage how an internal network reaches the 'Internet' (a Virtual LAN) by manipulating traffic paths using BGP attributes like Local Preference and AS-Path Prepending and to learn how to set up VRRP for redundancy. Bridging the physical Cisco 2921s to the virtual VyOS nodes was a significant hurdle, as documentation for building KVM-to-Physical bridges on Fedora OS is sparse. Additionally, I encountered a persistence issue where VyOS configurations would not save between reboots; rather than stopping, I used this as a challenge to become familiar with the CLI for the VyOS routers and the commands for setting up BGP.

## Network Topology

![Network_Topology.png](Network_Topology.png)

### Technical Stack

- **Physical Hardware:** 2x Cisco 2921 Routers (ISR), 1x Cisco Catalyst 2960 Switch.
- **Virtual Infrastructure:** 2x VyOS Routers (KVM), GNS3, Fedora Linux Bridge.
- **Protocols:** eBGP, iBGP, OSPF, VRRP, ICMP.
- **Traffic Routing:** BGP Route-Maps, Local Preference, AS-Path Prepending.

### Physical & Virtual Connectivity

- Cisco R1 (g0/0) ↔ Fedora Bridge (eth0) → VyOS R1 (eth1)
- Cisco R2 (g0/0) ↔ Fedora Bridge (eth1) → VyOS R2 (eth1)
- VyOS R1 (eth0) ↔ VyOS R2 (eth0) (Virtual iBGP/VRRP link)
- Cisco R1 (g0/1) ↔ Cisco R2 (g0/1) (Physical iBGP link)

### Key Features

- **Hybrid Peering:** Established eBGP peering between physical Cisco IOS and virtual VyOS nodes over a Linux KVM bridge.
- **Redundant Gateways:** Implemented **VRRP** on the VyOS internal interfaces to provide a single Virtual IP (192.168.10.1) for the Virtual LAN.
- **Deterministic Routing:** Applied **Local Preference** on Cisco R1 to force outbound traffic through VyOS R1 and **AS-Path Prepending** on VyOS R2 to influence inbound traffic.
- **Internal Sync:** Used **OSPF** and **iBGP** to ensure Cisco R1 and R2 maintain a synchronized routing table, allowing for a "scenic route" failover if a direct link is lost.

### IP Schema

| Device | Management IP | BGP AS | Role |
| --- | --- | --- | --- |
| Cisco R1 | 10.0.0.2 | 200 | Physical Edge A |
| Cisco R2 | 10.0.0.6 | 200 | Physical Edge B |
| VyOS R1 | 10.0.0.1 | 100 | Virtual Gateway (Primary) |
| VyOS R2 | 10.0.0.5 | 100 | Virtual Gateway (Secondary) |
| Virtual VIP | 192.168.10.1 | N/A | LAN Default Gateway |

## Configuration & Policies

#### **BGP Path Manipulation (Cisco IOS)**

This configuration ensures Cisco R1 prefers VyOS R1 for all traffic destined for the 192.168.10.0/24 network.

```bash
ip prefix-list PL-VIRT-LAN permit 192.168.10.0/24

route-map RM-PREFER-VIRT-1 permit 10
 match ip address prefix-list PL-VIRT-LAN
 set local-preference 200

router bgp 200
 neighbor 10.0.0.1 remote-as 100
 address-family ipv4
  neighbor 10.0.0.1 route-map RM-PREFER-VIRT-1 in
```

### **AS-Path Prepending (VyOS)**

This policy makes VyOS R2 look "further away" to external peers, ensuring it only receives traffic if VyOS R1 fails.

```bash
set policy prefix-list PL-VIRT-LAN rule 10 action 'permit'
set policy prefix-list PL-VIRT-LAN rule 10 prefix '192.168.10.0/24'

set policy route-map RM-AS-PREPEND rule 10 action 'permit'
set policy route-map RM-AS-PREPEND rule 10 match ip address prefix-list 'PL-VIRT-LAN'
set policy route-map RM-AS-PREPEND rule 10 set as-path-prepend '100 100 100'

set protocols bgp 100 neighbor 10.0.0.6 address-family ipv4-unicast route-map export 'RM-AS-PREPEND'
```

### High Availability Testing

- **Test Case:** Continuous ICMP ping from Cisco R1 to the Virtual LAN VIP (192.168.10.1).
- **Simulated Failure:** Administratively disabled `eth0` on VyOS R1 and eventually performed a hard "Power Off" of the VM.
- **Observed Behavior:**
    - **VRRP:** VyOS R2 transitioned from `BACKUP` to `MASTER` in <1 second.
    - **BGP:** Cisco R1 withdrew the 10.0.0.1 route and automatically rerouted traffic through the iBGP path via Cisco R2 → VyOS R2.
    - **Recovery:** Re-enabling VyOS R1 saw traffic automatically "fail-back" to the primary path due to the higher Local Preference and VRRP Priority.

## Lessons Learned & Technical Challenges

**1. The "Zombie Router"** 

- **Challenge:** Shutting down only the LAN interface of VyOS R1 caused the BGP session to stay "Up" even though traffic couldn't reach the destination.
- **Solution:** Identified the need for **Interface Tracking** in VRRP to ensure that if a LAN interface fails, the WAN-facing BGP session or VRRP priority is also dropped.

**2.  BGP Hold Timer Latency**

- **Challenge:** Initial failover took nearly 3 minutes because of the default BGP 180-second hold timer.
- **Solution:** Tuned BGP timers to `keepalive 10` and `holdtime 30` across all peers to achieve faster convergence in a lab environment.

**3.  Bridge Integration**

- **Challenge:** Traffic was unable to cross the bridge from VyOS to the physical Cisco routers.
- **Solution:** I discovered the `gns3tap0-1` interface needed to be manually added to the bridge. After verifying traffic reached the bridge with `tcpdump`, I found the physical switch port was still configured as an **Access Port (VLAN 10)** from a previous lab. Changing it to a **Trunk** allowed the multi-VLAN traffic to pass.

---
