# **HOMELAB NETWORK** 

My Homelab network is built in order for me to learn and gain hands on experience on networking and how different networking devices interact with one another and through making this homelab I was able to gain more control and deeper understanding on data moves through different components and its uses.

## Skills Demonstrated

- VLAN design and implementation
- IEEE 802.1Q trunking
- Router-on-a-Stick architecture
- Inter-VLAN routing
- DNS administration (Pi-hole)
- Network segmentation
- Remote access VPN (Tailscale)
- Switch configuration and management
- Cisco IOS CLI
- Network documentation
- Troubleshooting and validation

## Network Topology 

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

## Physical Homelab

### Full Setup




## Configuration

### Cisco ISR4321
For the router I used the router on a stick method for Inter VLAN routing and added extra security through adding a password when accessing priveleged exec mode.
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


### IP Interface 

![IP int Diagram](https://github.com/Edualk12/homelab-network/blob/main/ip%20int.png)


## How It Operates 

### Traffic Flow 
Traffic enters from the ISP modem to the G0/0/0 interface of the Cisco ISR4321 Router and enters the network though the G0/0/1 interface 

### VLAN Routing
The Cisco ISR4321 uses router-on-a-stick via a 
single physical interface (GigabitEthernet0/0/1) 
with dot1Q subinterfaces for each VLAN:

```cisco
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
```

### Switching
The EnGenius EWS7928P connects to the Cisco router and it acts as the default gateway
via gi1 as a tagged trunk carrying VLANs 10, 20 
and 30. Access ports are assigned per VLAN:

- gi3, gi7, gi21 → VLAN 10 (main)
- gi19 → VLAN 20
- gi5 → VLAN 30 (guest)

### DHCP
Each VLAN has its own DHCP pool on the Cisco router.
Where the interface address of the cisco router is excluded. 

```ip domain name KlaudeRouter
ip dhcp excluded-address 192.168.1.1
ip dhcp excluded-address 192.168.2.1
ip dhcp excluded-address 192.168.3.1
ip dhcp excluded-address 192.168.4.1
!
```
DHCP pool of each vlan, where only in VLAN10 the primary DNS-server is set to the Pihole PC.
```
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
```

### DNS
The HP Thin Client where its connected via VLAN 10 uses Pi-hole on 192.168.1.69 as primary DNS.
The Cisco ISR4321 also acts as the secondary or backup DNS incase the HP Thin Clients Fails somehow.

### NAT
All four subnets NAT out through GigabitEthernet0/0/0 
via PAT overload on the Cisco router.

### Security 
Using the Cisco ISR4321 onboard router for the firewall and the Pihole Ad Blocker and DNS server used for filtering.

## Remote Access via Tailscale VPN

Due to CGnat which my IP is dynamic I am not able to securely and remotely connect to my network using traditional methods such as Wireguard.


![VPN Diagram](https://github.com/Edualk12/homelab-network/blob/main/VPN.png)

A quick and simple alternative I found is through using tailscale to access my network through their relay servers to bypass the dynamic IP address.

I installed tailscale to my main server and main PC using the HP Thin Client server as the main server to wake up the other computers using Wake-on-Lan through SSH clients (Termius, MobaXterm)

## Pi-hole DNS Server | Ad Blocker

### Overview

Benefits:
- Centralized DNS management
- Advertisement blocking
- Reduced malicious domain exposure
- Faster repeat DNS lookups through caching

Pi-hole runs on the HP Thin Client (T530) on VLAN 10 at a 
static IP of `192.168.1.69`, serving as the primary DNS 
server for the main network.

![Pihole Diagram](https://github.com/Edualk12/homelab-network/blob/main/pihole.png)

### Why Only VLAN 10?
Pi-hole is intentionally scoped to VLAN 10 only. VLAN 20 
and VLAN 30 (guest/wifi) use Google DNS (8.8.8.8) directly 
to avoid blocking legit traffic for other users and 
to keep the guest network isolated from internal DNS.

### DHCP DNS Assignment
The Cisco ISR4321 assigns Pi-hole as primary DNS for VLAN 10:


## Validation

### Inter-VLAN Routing

PC on VLAN 10 successfully pinged:
- VLAN 10 Gateway 192.168.1.1
  
![ping10 Diagram](https://github.com/Edualk12/homelab-network/blob/main/ping%20vlan%2010.png)


- VLAN 20 Gateway 192.168.2.1
  
![ping20 Diagram](https://github.com/Edualk12/homelab-network/blob/main/ping%20vlan%2020.png)
  
- VLAN 30 Gateway 192.168.3.1
  
![ping30 Diagram](https://github.com/Edualk12/homelab-network/blob/main/ping%20vlan%2030.png)

### DNS

Pi-hole successfully resolves DNS requests from VLAN 10.

## What I Learned
- Router-on-a-stick with dot1Q subinterfaces for VLAN
- How different brands handle configuration (Cisco, Engenius)
- Inter-VLAN routing and DHCP pool for each VLAN
- Importance of excluding gateway IPs from DHCP
- improving security through enabling SSH and disabling telnet on all devices
- How Pi-hole integrates as primary DNS per VLAN
- Cisco IOS-XE software licensing limitations

## What I'd Improve (Work in Progress)
- Add MikroTik as perimeter firewall
- Implement inter-VLAN ACLs to isolate guest traffic
- Move routing to MikroTik to bypass 50Mbps cap
- Add Zabbix SNMP monitoring for all devices
- implementing ddns server

  
