# **HOMELAB NETWORK** 

My Homelab network is built in order for me to learn and gain hands on experience on networking and how different networking devices interact with one another and through making this homelab I was able to gain more control and deeper understanding on data moves through different components and its uses.

## Network Topology 🗺️

![Network Diagram](https://github.com/Edualk12/HOMELAB-NETWORK/blob/main/HOMELAB%20V4.png)


##  Hardware

### Network Devices
| Device | Model | Role |
|--------|-------|------|
| Router | Cisco ISR4321 | Inter-VLAN routing, DHCP Server, NAT |
| Firewall (WIP) | MikroTik (WIP) | Perimeter firewall |
| Switch | EnGenius EWS7928P | Managed switching |
| Wireless AP | EnGenius ews300ap | Wi-Fi |

### Servers / Computer
| Device | CPU | RAM | Storage | OS/Role |
|--------|-----|-----|---------|------|
| HP Thin Client T530 | AMD Embedded G-Series GX-215JJ | 4GB | 256GB | Ubuntu Server (Runs Pihole 24/7, Main DNS Server) |
| Dell Optiplex 3050 Micro | Intel Core I5-8500T | 16GB | 256GB | Proxmox VE (Testing Environment) |

## VLAN Design
| VLAN ID | Name | Subnet | Device Connected |
|---------|------|--------|------------------|
| 10 | klaude | 192.168.1.0/24 | Klaude PC, HP Thin Client |
| 20 | kamange | 192.168.2.0/24 | Kamange PC, Guest PC |
| 30 | others | 192.168.3.0/24 | Engenius AP, WIFI |

## Configuration

### Cisco ISR4321
For the router I used the router on a stick method for Inter VLAN routing and added extra security through adding a password when accessing privelaged exec mode.
```!
! KlaudeRouter - Cisco ISR4321
! Last modified: Jun 2 2026
!
version 16.9
service timestamps debug datetime msec
service timestamps log datetime msec
platform qfp utilization monitor load 80
no platform punt-keepalive disable-kernel-core
!
hostname KlaudeRouter
!
vrf definition Mgmt-intf
 !
 address-family ipv4
 exit-address-family
 !
 address-family ipv6
 exit-address-family
!
enable secret <REDACTED>
!
no aaa new-model
!
ip domain name KlaudeRouter
ip dhcp excluded-address 192.168.1.1
ip dhcp excluded-address 192.168.2.1
ip dhcp excluded-address 192.168.3.1
ip dhcp excluded-address 192.168.4.1
!
ip dhcp pool VLAN10
 network 192.168.1.0 255.255.255.0
 dns-server 192.168.1.69 8.8.8.8
 domain-name klaudeVlan10
 default-router 192.168.1.1
 lease infinite
!
ip dhcp pool VLAN20
 network 192.168.2.0 255.255.255.0
 dns-server 8.8.8.8
 default-router 192.168.2.1
 domain-name klaudeVlan20
 lease infinite
!
ip dhcp pool VLAN30
 network 192.168.3.0 255.255.255.0
 dns-server 8.8.8.8
 default-router 192.168.3.1
 domain-name klaudeVlan30
 lease infinite
!
ip dhcp pool DEFAULTVLAN
 network 192.168.4.0 255.255.255.0
 dns-server 8.8.8.8
 default-router 192.168.4.1
 domain-name klaudeDefaultVlan
 lease infinite
!
subscriber templating
multilink bundle-name authenticated
!
license udi pid ISR4321/K9 sn <REDACTED>
no license smart enable
diagnostic bootup level minimal
!
spanning-tree extend system-id
!
username <admin> secret <REDACTED>
!
redundancy
 mode none
!
interface GigabitEthernet0/0/0
 ip address dhcp
 ip nat outside
 negotiation auto
!
interface GigabitEthernet0/0/1
 no ip address
 ip nat inside
 negotiation auto
!
interface GigabitEthernet0/0/1.1
 encapsulation dot1Q 10
 ip address 192.168.1.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1.2
 encapsulation dot1Q 20
 ip address 192.168.2.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1.3
 encapsulation dot1Q 30
 ip address 192.168.3.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0/0/1.4
 encapsulation dot1Q 1 native
 ip address 192.168.4.1 255.255.255.0
 ip nat inside
!
interface GigabitEthernet0
 vrf forwarding Mgmt-intf
 no ip address
 shutdown
 negotiation auto
!
ip nat inside source list 1 interface GigabitEthernet0/0/0 overload
ip forward-protocol nd
ip http server
ip http authentication local
ip http secure-server
ip tftp source-interface GigabitEthernet0
!
ip ssh version 2
!
access-list 1 permit 192.168.1.0 0.0.0.255
access-list 1 permit 192.168.2.0 0.0.0.255
access-list 1 permit 192.168.3.0 0.0.0.255
access-list 1 permit 192.168.4.0 0.0.0.255
!
control-plane
!
line con 0
 login local
 transport input none
 stopbits 1
line aux 0
 stopbits 1
line vty 0 4
 exec-timeout 30 0
 login local
 transport input ssh
line vty 5 97
 exec-timeout 30 0
 login local
 transport input ssh
!
end
```

### Engenius EWS7928P
I configured the switch using a serial to USB converter and used PuTTy terminal to access the switch's terminal, I assigned the gi1 port as the trunk port or hybrid port in this case that is connected to the cisco router itself and assigned different ports to different VLANs as shown below in the config.
```! EWS7928P - EnGenius 28-Port Gigabit Switch
! Firmware: v1.05.45-c1.8.57
!
ipv6 state autoconfig
username "admin" secret encrypted <REDACTED>
username "klaude" secret encrypted <REDACTED>

vlan 1
 name "default"
vlan 10
 name "klaude"
vlan 20
 name "Kamange"
vlan 30
 name "others"

spanning-tree mst configuration
 name "<REDACTED>"
!
snmp community private rw <REDACTED>
snmp community public ro <REDACTED>
snmp engineid <REDACTED>
!
no ip telnet
ip ssh
!
interface gi1
 switchport mode hybrid
 switchport hybrid allowed vlan add 10,20,30 tagged
 ! Trunk port to Cisco ISR4321 - carries all VLANs tagged
!
interface gi3
 switchport mode hybrid
 switchport hybrid pvid 10
 switchport hybrid allowed vlan add 10 untagged
 ! Access port - VLAN 10
!
interface gi5
 switchport mode hybrid
 switchport hybrid pvid 30
 switchport hybrid allowed vlan add 30 untagged
 switchport hybrid allowed vlan remove 1
 ! Access port - VLAN 30, removed from default VLAN
!
interface gi7
 switchport mode hybrid
 switchport hybrid pvid 10
 switchport hybrid allowed vlan add 10 untagged
 switchport hybrid allowed vlan remove 1
 ! Access port - VLAN 10
!
interface gi19
 switchport mode hybrid
 switchport hybrid pvid 20
 switchport hybrid allowed vlan add 20 untagged
 switchport hybrid allowed vlan remove 1
 ! Access port - VLAN 20
!
interface gi21
 switchport mode hybrid
 switchport hybrid pvid 10
 switchport hybrid allowed vlan add 10 untagged
 switchport hybrid allowed vlan remove 1
 ! Access port - VLAN 10
!
interface gi25
 switchport mode hybrid
 speed auto duplex full
!
interface gi26
 switchport mode hybrid
 speed auto duplex full
!
interface gi27
 switchport mode hybrid
 speed auto duplex full
!
interface gi28
 switchport mode hybrid
 speed auto duplex full
!
```

## How It Operates 


