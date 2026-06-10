# **HOMELAB NETWORK** 

My Homelab network is built in order for me to learn and gain hands on experience on networking and how different networking devices interact with one another and through making this homelab I was able to gain more control and deeper understanding on data moves through different components and its uses.


## Network Topology 

![Network Diagram](https://github.com/Edualk12/HOMELAB-NETWORK/blob/main/HOMELAB%20V4.png)


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


![Real Photo](https://github.com/Edualk12/homelab-network/blob/main/homelab%20pic.jpg)



## Configuration

### Cisco ISR4321
For the router I used the router on a stick method for Inter VLAN routing and added extra security through adding a password when accessing priveleged exec mode.

Full configuration: [config/cisco-isr4321.cfg](config/cisco-isr4321.cfg)


### Engenius EWS7928P
I configured the switch using a serial to USB converter and used PuTTy terminal to access the switch's terminal, I assigned the gi1 port as the trunk port or hybrid port in this case that is connected to the cisco router itself and assigned different ports to different VLANs as shown below in the config.

Full configuration: [config/engenius-ews7928.cfg](config/engenius-ews7928.cfg)


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

  
