# Template repo for SONiC PoC Repos

[![Contributions welcome](https://img.shields.io/badge/contributions-welcome-orange.svg)](#-how-to-contribute)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/Dell-Networking/PoC-SONiC-template/blob/main/LICENSE.md)
[![GitHub issues](https://img.shields.io/github/issues/Dell-Networking/PoC-SONiC-template)](https://github.com/Dell-Networking/PoC-SONiC-template/issues)

Built and maintained by [Ben Goldstone](https://github.com/benjamingoldstone/) and [Contributors](https://github.com/Dell-Networking/PoC-SONiC-template/graphs/contributors)

------------------

**Proof of Concept for Network Function Chaining**
Q-in-Q VLAN tunneling allows service providers to separate VLAN traffic from different customers by tunneling multiple VLANs from one customer (CVLANs) in a single, customer-specific service-provider VLAN (SVLAN). When the customer traffic enters the service provider network, a second 802.1Q tag is added to a customer-tagged frame. The encapsulated packet consists of an inner CVLAN tag of the private customer network and an outer SVLAN tag of the public provider network.

Q-in-Q VLAN tunneling is supported on physical interfaces — Ethernet and port channels. Q-in-Q tunneling is also known as VLAN stacking and complies with the 802.1ad standard.

If the provider network uses a VXLAN overlay (see VXLAN), the customer VLAN traffic that is identified by an SVLAN is mapped to a VXLAN network identifier (VNI) and forwarded based on the VNI. The SVLAN header is replaced with a VNI. Each unique VNI maintains L2 isolation (separate bridging domains) from other customer tenant segments. Before egress to a customer network, the VNI in VLAN packets is mapped to the SVLAN. Forwarding decisions are made by a Provider Edge switch based on the SVLAN.

## Contents

- [Description and Objective](#-description-and-objective)
- [Topology Diagram](#-Topology)
- [Requirements](#-requirements)
- [Example](#-Sample)
- [Planned Enhancements](#-Future)
- [How to Contribute](#-how-to-contribute)

## 🚀 Description and Objective

Network Service Chaining (NFC) is a capability to create a chain of connected network services, such as Firewall, Network Monitoring, VPN, Intruder Prevention, etc. leveraging SDN programmability. SR-IOV is a hardware standard that allows a PCI Express device – typically a network interface card (NIC) – to present itself as several virtual NICs to a hypervisor. In this example, a single SR-IOV NIC can scale up to 128 VMs passing traffic via same switch port. This allows Service Provider to offer revenue generating services with minimum CAPEX. The traffic can also flow via VXLAN in between VNFs. Packet received from VNF1 on Port P1 on switch SW1 can be mapped via translation rule to send to VNF2 on the same port P1. 

## 🚀 Topology

![image](https://github.com/Dell-Networking/PoC-Network-Function-Chaining-with-DES/assets/18683207/8d0566f4-4f7c-4380-be2f-80738999b306)

![image](https://github.com/Dell-Networking/PoC-Network-Function-Chaining-with-DES/assets/18683207/1eb2a9c6-7b2a-449a-becf-b3b7504a3e8c)

## 📋 Requirements

SRIOV enabled NICs
Dell Enterprise SONiC 4.2

## 📋 Sample

In the above diagram, Server-3 can host multiple VNFs and is connected to leaf switch-16. Similarly, Server-5 can also host multiple VNFs and is connected to leaf switch-15. To stretch L2 segments across an IP core, switch-12 provides Spine connectivity for BGP EVPN VXLAN fabric. The encapsulation and decapsulation of VXLAN headers is handled by hardware VTEP. 
Servers Configuration
SRIOV is enabled on network interface card (eno1) with the below script to define 4 VF (virtual functions) associated with the PCIe PF (physical function) on Server-3 and Server-5. 
poc@server3:~$ cat net-config.sh | grep eno1
# Configure maximum number of VFs on eno1 and eno2
echo 4 > /sys/class/net/eno1/device/sriov_numvfs
ip link set dev eno1 vf 0 vlan 100
ip link set dev eno1 vf 1 vlan 101
ip link set dev eno1 vf 2 vlan 102
ip link set dev eno1 vf 3 vlan 103
ip link set eno1 up
ip link set eno1v0 netns chain1-a
ip link set eno1v1 netns chain1-vnf0
ip netns exec chain1-a ip link set eno1v0 up
ip netns exec chain1-vnf0 ip link set eno1v1 up
ip netns exec chain1-a ip addr add 10.1.1.1/24 dev eno1v0
ip netns exec chain1-vnf0 ip link add link eno1v1 name eno1v1.10 type vlan id 10
ip netns exec chain1-vnf0 ip link set eno1v1.10 up
ip netns exec chain1-vnf0 ip addr add 10.1.2.1/24 dev eno1v1.10

use ip netns list statement to show all the named network namespaces on the Linux machine. 
poc@server3:~$ ip netns list
chain1-vnf0 (id: 1)
chain1-a (id: 0)

poc@server5:~$ ip netns list
chain1-vnf1 (id: 1)
chain1-z (id: 0)

ip link show shows the state of all network interfaces on the system. Specify network interface ID (eno1 in the below example) to show state of a particular interface. In the below output, eno1 has 4 virtual functions define (vf0, vf1, vf2 and vf3). Each vf has a unique mac-address and VLAN assignment. 
poc@server3:~$ ip link show eno1
3: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 24:6e:96:5b:cd:8e brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether 06:e4:59:d0:97:00 brd ff:ff:ff:ff:ff:ff, vlan 100, spoof checking on, link-state auto, trust off
    vf 1     link/ether a2:c2:2b:2d:e1:95 brd ff:ff:ff:ff:ff:ff, vlan 101, spoof checking on, link-state auto, trust off
    vf 2     link/ether ae:c6:74:e8:3a:83 brd ff:ff:ff:ff:ff:ff, vlan 102, spoof checking on, link-state auto, trust off
    vf 3     link/ether 02:ab:8b:b0:4f:c0 brd ff:ff:ff:ff:ff:ff, vlan 103, spoof checking on, link-state auto, trust off
    altname enp25s0f0

poc@server5:~$ ip link show eno1
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 24:6e:96:96:2f:64 brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether ee:c4:e5:c0:c6:05 brd ff:ff:ff:ff:ff:ff, vlan 300, spoof checking on, link-state auto, trust off
    vf 1     link/ether 2a:69:0a:d4:a3:79 brd ff:ff:ff:ff:ff:ff, vlan 301, spoof checking on, link-state auto, trust off
    vf 2     link/ether 86:f9:78:2d:17:13 brd ff:ff:ff:ff:ff:ff, vlan 302, spoof checking on, link-state auto, trust off
    vf 3     link/ether a6:48:bc:4a:50:b2 brd ff:ff:ff:ff:ff:ff, vlan 303, spoof checking on, link-state auto, trust off
    altname enp25s0f0

poc@server5:~$ sudo ip netns exec chain1-z ip addr list
[sudo] password for poc:
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: eno1v0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether ee:c4:e5:c0:c6:05 brd ff:ff:ff:ff:ff:ff
    altname enp25s0f0v0
    inet 10.1.3.1/24 scope global eno1v0
       valid_lft forever preferred_lft forever
    inet6 fe80::ecc4:e5ff:fec0:c605/64 scope link
       valid_lft forever preferred_lft forever

Verify the connectivity with ping test across the fabric:
poc@server3:~$ sudo ip netns exec chain1-a ping 10.1.3.1
PING 10.1.3.1 (10.1.3.1) 56(84) bytes of data.
64 bytes from 10.1.3.1: icmp_seq=1 ttl=62 time=0.418 ms
64 bytes from 10.1.3.1: icmp_seq=2 ttl=62 time=0.481 ms

Switch Configuration
The switches are configured to process Double-Tagged, Single-Tagged and Untagged packets. 
On switch-16, configure Q-in-Q VLAN tunneling or VLAN translation for transmitting customer VLAN traffic over a service provider network. In this example, VLAN 100 (CVLAN) is mapped to VLAN 2000 (SVLAN). Likewise, VLAN 101 is configured for double-tagged packets with inner tag of 10 AND outer tag of 2100. 
interface Eth1/1
 mtu 9100
 speed 10000
 unreliable-los auto
 no shutdown
 switchport trunk allowed Vlan 102
 switchport vlan-mapping 100 2000
 switchport vlan-mapping 101 inner 10 2100

VLAN 200 is also double tagging the packet with inner tag of 10 and outer tag of 2100
interface Eth1/2
 mtu 9100
 speed 10000
 unreliable-los auto
 no shutdown
 switchport vlan-mapping 200 inner 10 2000
!
For fabric wide connectivity, configure VNI-VLAN and/or VNI-VRF mappings 
interface vxlan vtep0
 source-ip 192.168.11.2
 external-ip 192.168.12.2
 vni-downstream external
 qos-mode pipe dscp 0
 map vni 1000102 vlan 102
 map vni 1002000 vlan 2000
 map vni 1002100 vlan 2100

On switch-15, VLAN302 and VLAN303 are customer VLANs.
interface Vlan302
 neigh-suppress
 ip vrf forwarding Vrf-CustomerA
 ip anycast-address 10.11.11.1/24
!
interface Vlan303
 neigh-suppress
 ip vrf forwarding Vrf-CustomerB
 ip anycast-address 10.11.11.1/24

VLAN 300 is mapped to 2200 whereas VLAN301 is double-tagged. It has an inner tag of 10 and outer tag of 2300. 
interface Eth1/1
 mtu 9100
 speed 10000
 unreliable-los auto
 no shutdown
 switchport trunk allowed Vlan 302-303
 switchport vlan-mapping 300 2200
 switchport vlan-mapping 301 inner 10 2300
 service-policy type qos in VLAN-TEST

Interface Eth1/2 has an untagged traffic
interface Eth1/2
 mtu 9100
 speed 10000
 unreliable-los auto
 no shutdown
 switchport access Vlan 2200

VNI-VLAN mapping is for fabric wide connectivity:
interface vxlan vtep0
 source-ip 192.168.11.1
 external-ip 192.168.12.1
 vni-downstream external
 qos-mode pipe dscp 0
 map vni 1000302 vlan 302
 map vni 1000303 vlan 303
 map vni 1002000 vlan 2000
 map vni 1002100 vlan 2300
!
Use various show commands to verify the topology:
switch15-s5248# show lldp table
-----------------------------------------------------------------------------------------------------------------
LocalPort     RemoteDevice        RemotePortID        Capability      RemotePortDescr
-----------------------------------------------------------------------------------------------------------------                                   
Eth1/1         server5            24:6e:96:96:2f:64       R               eno1
Eth1/49       switch12-z9264      Eth1/62                 R               Ethernet244
Eth1/56       switch16-s5248      Eth1/56                 R               Ethernet76

switch16-s5248# show lldp table
---------------------------- -----------------------------------------------------------------------------------
LocalPort     RemoteDevice        RemotePortID        Capability      RemotePortDescr
----------------------------------------------------------------------------------------------------------------
Eth1/1        server3             24:6e:96:5b:cd:8e       BR            eno1
Eth1/2        server3             24:6e:96:5b:cd:90       BR            eno2
Eth1/49       switch12-z9264      Eth1/60                 R             Ethernet236
Eth1/56       switch15-s5248      Eth1/56                 R             Ethernet76

switch15-s5248# show Vlan
Q: A - Access (Untagged), T - Tagged
NUM        Status      Q Ports                           Autostate         Dynamic
300        Inactive                                       Enable
302        Active      T  Eth1/1                          Enable             No
                       A  Vxlan_192.168.12.2                                 No
303        Active      T  Eth1/1              Enable                         No
                       A  Vxlan_192.168.12.2                                 No
1501       Inactive                                        Enable
2000       Active      A  Vxlan_192.168.12.2 Disable                         No
2100       Inactive                                        Enable
2200       Active      T  Eth1/1                           Enable            No
                       A  Eth1/2                                             No
2300       Active      T  Eth1/1                           Enable            No
                       A  Vxlan_192.168.12.2                                 No

switch16-s5248# show vlan 2000
Q: A - Access (Untagged), T - Tagged
NUM        Status      Q Ports                            Autostate          Dynamic
2000       Active      T  Eth1/1                           Disable             No
                       T  Eth1/2                                               No
                       A  Vxlan_192.168.12.1                                   No

switch16-s5248# show vlan 2100
Q: A - Access (Untagged), T - Tagged
NUM        Status      Q Ports                             Autostate         Dynamic
2100       Active      T  Eth1/1           Enable                               No
                       A  Vxlan_192.168.12.1                                    No

switch15-s5248# show interface vlan-mappings
Flags: M - Multi-tag
---------------------------------------------------------
Name            Outer  Inner  Mapped Vlan  Priority Flags
-----------------------------------------------------------
Eth1/1          300     -         2200       -        -
Eth1/1          301     10        2300       -        -
switch15-s5248#

switch16-s5248# show interface vlan-mappings
Flags: M - Multi-tag
--------------------------------------------------------------------------
Name            Outer  Inner  Mapped Vlan  Priority Flags
--------------------------------------------------------------------------
Eth1/1          100       -       2000       -        -
Eth1/1          101      10       2100       -        -
Eth1/2          200      10       2000       -        -

## 👏 Future

•	Add configuration snippet for 3rd party network functions such as virtualized routers, load balancers, firewalls, SD-WAN, etc. 
•	Add SR-IOV integration with OpenStack Neutron to automatically create corresponding ports when a network is created
•	Automation scripts for testing and validation

## 👏 How to Contribute

We welcome contributions to the project. Please reference the [CONTRIBUTING](https://github.com/Dell-Networking/PoC-Index/blob/main/CONTRIBUTING.md) guide in the PoC-Index repo for more details (this guide is common across Dell Networking PoC projects).



