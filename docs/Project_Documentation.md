# Network-1 Project Documentation

## 1. Network Design & Topology Overview
The network topology represents a Wide Area Network (WAN) interconnecting four distinct sites/routers:
*   **Admin Center Router (Hub):** Acts as the central hub connecting to all other routers, providing central services like DHCP. It also connects to the ISP and performs NAT to the outside world.
*   **Engineer Router:** Connects the Engineering department, which is segmented into multiple VLANs (Staff, Labs, and a Server network). 
*   **Science Router:** Connects the Science department LAN.
*   **Art Router:** Connects the Art department LAN.
*   **ISP Router:** Represents the Internet Service Provider, terminating the outside public connection.

This hub-and-spoke-like design uses Static Routing for inter-network reachability, with Site-to-Site VPN configured between the Engineer and Science branches for secure data transmission.

---

## 2. VLSM Addressing Table
Based on the configuration files, the IP addressing scheme utilizes Variable Length Subnet Masking (VLSM).

| Location / Network | Network Address | Subnet Mask | First Usable IP | Last Usable IP | Broadcast Address | Default Gateway (Router IP) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Engineer - Labs (VLAN 20)** | 192.168.10.0 | 255.255.255.128 (/25) | 192.168.10.1 | 192.168.10.126 | 192.168.10.127 | 192.168.10.1 |
| **Engineer - Staff (VLAN 10)** | 192.168.10.128 | 255.255.255.128 (/25) | 192.168.10.129 | 192.168.10.254 | 192.168.10.255 | 192.168.10.129 |
| **Science LAN** | 192.168.11.0 | 255.255.255.128 (/25) | 192.168.11.1 | 192.168.11.126 | 192.168.11.127 | 192.168.11.1 |
| **Art LAN** | 192.168.11.128 | 255.255.255.192 (/26) | 192.168.11.129 | 192.168.11.190 | 192.168.11.191 | 192.168.11.129 |
| **Engineer - Servers (VLAN 30)**| 192.168.11.192 | 255.255.255.248 (/29) | 192.168.11.193 | 192.168.11.198 | 192.168.11.199 | 192.168.11.193 |
| **WAN: Admin - Engineer** | 10.0.0.0 | 255.255.255.248 (/29) | 10.0.0.1 | 10.0.0.6 | 10.0.0.7 | N/A |
| **WAN: Admin - Science** | 10.0.0.8 | 255.255.255.252 (/30) | 10.0.0.9 | 10.0.0.10 | 10.0.0.11 | N/A |
| **WAN: Admin - Art** | 10.0.0.12 | 255.255.255.252 (/30) | 10.0.0.13 | 10.0.0.14 | 10.0.0.15 | N/A |
| **WAN: Admin - ISP** | 200.0.0.0 | 255.255.255.248 (/29) | 200.0.0.1 | 200.0.0.6 | 200.0.0.7 | N/A |

---

## 3. Protocol Implementation Details

### DHCP (Dynamic Host Configuration Protocol)
The **Admin Router** acts as the central DHCP server for the entire network. Branch routers use the `ip helper-address` command on their LAN-facing interfaces to forward DHCP broadcasts to the Admin Router.
*   **Pool `lan1v10`**: Provides IPs for Engineer Staff VLAN 10 (192.168.10.128/25).
*   **Pool `lan1v20`**: Provides IPs for Engineer Labs VLAN 20 (192.168.10.0/25).
*   **Pool `lan2`**: Provides IPs for Science LAN (192.168.11.0/25).
*   **Pool `lan3`**: Provides IPs for Art LAN (192.168.11.128/26).

### DNS & Web Server
*   All DHCP clients are assigned `192.168.11.194` as their DNS Server. This server is located in the Engineer Server network (VLAN 30).
*   Static NAT exposes the internal Web/DNS servers to the outside.
*   **Note**: The packet tracer configuration contains the network layout for the DNS/Web servers but they are managed via the end-device interface directly rather than router CLI.

### VLAN (Virtual LAN)
VLANs are implemented primarily in the Engineer department to segment broadcast domains:
*   **VLAN 10**: Staff Network.
*   **VLAN 20**: Labs Network.
*   **VLAN 30**: Servers Network.
*   **Inter-VLAN Routing**: Router-on-a-Stick is configured on the Engineer Router using sub-interfaces (`GigabitEthernet0/0/0.10`, `.20`, `.30`) with `encapsulation dot1Q`.
*   Switches (`Switch_access_eng`, `Switch_core_eng`, `Switch_denstination_eng`) are configured with appropriate Access ports for endpoints and Trunk ports for switch-to-switch and switch-to-router connections.

### NAT (Network Address Translation)
NAT is configured on the **Admin Router** to translate private internal IPs to public IPs when traversing the ISP.
*   **PAT (Port Address Translation / Overload)**: `ip nat inside source list 1 interface Serial0/2/1 overload` allows all internal networks (permitted by ACL 1) to access the internet using the Admin Router's Serial0/2/1 IP address.
*   **Static NAT**: `ip nat inside source static 192.168.11.194 200.0.0.3` and `192.168.11.195 200.0.0.4` map the internal DNS and Web servers to public IP addresses, making them accessible from the outside network.

### VPN (Site-to-Site IPsec)
A Site-to-Site IPsec VPN tunnel is established between the **Engineer Router** and the **Science Router**.
*   **ISAKMP Policy**: Group 2, AES Encryption, Pre-Shared Key (`cisco`).
*   **Transform Set**: `ts` using `esp-aes` and `esp-sha-hmac`.
*   **Crypto Map**: `VPN` is applied to the WAN-facing serial interfaces.
*   **Interesting Traffic**: Defined by ACL 110. Traffic between the Engineer network (`192.168.11.192/29`) and Science network (`192.168.11.0/25`) triggers the encryption.

### Static Routing
All routers use Static Routing (`ip route`) to reach remote networks.
*   **Admin Router** has static routes pointing to the LAN networks behind the Engineer, Science, and Art routers.
*   **Branch Routers** (Engineer, Science, Art) have static routes pointing back to the Admin Router for inter-branch reachability.

---

## 4. Device Configurations

Below are the complete `running-config` configurations for each networking device in the project.

### Admin Router
```text
!
version 15.1
hostname Router
!
ip dhcp pool lan1v10
 network 192.168.10.128 255.255.255.128
 default-router 192.168.10.129
 dns-server 192.168.11.194
ip dhcp pool lan1v20
 network 192.168.10.0 255.255.255.128
 default-router 192.168.10.1
 dns-server 192.168.11.194
ip dhcp pool lan2
 network 192.168.11.0 255.255.255.128
 default-router 192.168.11.1
 dns-server 192.168.11.194
ip dhcp pool lan3
 network 192.168.11.128 255.255.255.192
 default-router 192.168.11.129
 dns-server 192.168.11.194
!
interface Serial0/2/0
 ip address 10.0.0.13 255.255.255.252
 ip nat inside
!
interface Serial0/2/1
 ip address 200.0.0.1 255.255.255.248
 ip nat outside
!
interface Serial0/3/0
 ip address 10.0.0.1 255.255.255.248
 ip nat inside
!
interface Serial0/3/1
 ip address 10.0.0.9 255.255.255.252
 ip nat inside
!
ip nat inside source list 1 interface Serial0/2/1 overload
ip nat inside source static 192.168.11.194 200.0.0.3 
ip nat inside source static 192.168.11.195 200.0.0.4 
ip classless
ip route 192.168.10.0 255.255.255.128 10.0.0.2 
ip route 192.168.10.128 255.255.255.128 10.0.0.2 
ip route 192.168.11.0 255.255.255.128 10.0.0.10 
ip route 192.168.11.128 255.255.255.192 10.0.0.14 
ip route 192.168.11.192 255.255.255.248 10.0.0.2 
!
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 1 permit 192.168.11.0 0.0.0.127
access-list 1 permit 192.168.11.128 0.0.0.63
!
end
```

### Engineer Router
```text
!
version 15.4
hostname Router
!
crypto isakmp policy 10
 encr aes
 authentication pre-share
 group 2
!
crypto isakmp key cisco address 10.0.0.10
!
crypto ipsec transform-set ts esp-aes esp-sha-hmac
!
crypto map VPN 10 ipsec-isakmp 
 set peer 10.0.0.10
 set transform-set ts 
 match address 110
!
interface GigabitEthernet0/0/0
 no ip address
 ip helper-address 10.0.0.1
 ip nat inside
 duplex auto
 speed auto
!
interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.129 255.255.255.128
 ip helper-address 10.0.0.1
 ip nat inside
!
interface GigabitEthernet0/0/0.20
 encapsulation dot1Q 20
 ip address 192.168.10.1 255.255.255.128
 ip helper-address 10.0.0.1
 ip nat inside
!
interface GigabitEthernet0/0/0.30
 encapsulation dot1Q 30
 ip address 192.168.11.193 255.255.255.248
 ip nat inside
!
interface Serial0/1/0
 ip address 10.0.0.2 255.255.255.248
 ip nat outside
 clock rate 2000000
 crypto map VPN
!
ip classless
ip route 10.0.0.8 255.255.255.252 10.0.0.1 
ip route 10.0.0.12 255.255.255.252 10.0.0.1 
ip route 192.168.11.0 255.255.255.128 10.0.0.1 
ip route 192.168.11.128 255.255.255.192 10.0.0.1 
ip route 200.0.0.0 255.255.255.248 10.0.0.1 
!
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.11.192 0.0.0.7 192.168.11.0 0.0.0.127
!
end
```

### Science Router
```text
!
version 15.4
hostname Router
!
crypto isakmp policy 10
 encr aes
 authentication pre-share
 group 2
!
crypto isakmp key cisco address 10.0.0.2
!
crypto ipsec transform-set ts esp-aes esp-sha-hmac
!
crypto map VPN 10 ipsec-isakmp 
 set peer 10.0.0.2
 set transform-set ts 
 match address 110
!
interface GigabitEthernet0/0/0
 ip address 192.168.11.1 255.255.255.128
 ip helper-address 10.0.0.9
 ip nat inside
 duplex auto
 speed auto
!
interface Serial0/1/0
 ip address 10.0.0.10 255.255.255.252
 ip nat outside
 clock rate 2000000
 crypto map VPN
!
ip classless
ip route 10.0.0.0 255.255.255.248 10.0.0.9 
ip route 10.0.0.12 255.255.255.252 10.0.0.9 
ip route 192.168.10.0 255.255.255.128 10.0.0.9 
ip route 192.168.10.128 255.255.255.128 10.0.0.9 
ip route 192.168.11.128 255.255.255.192 10.0.0.9 
ip route 200.0.0.0 255.255.255.248 10.0.0.9 
ip route 192.168.11.192 255.255.255.248 10.0.0.9 
!
access-list 1 permit 192.168.11.0 0.0.0.127
access-list 110 permit ip 192.168.11.0 0.0.0.127 192.168.11.192 0.0.0.7
!
end
```

### Art Router
```text
!
version 15.1
hostname Router
!
interface GigabitEthernet0/0
 ip address 192.168.11.129 255.255.255.192
 ip helper-address 10.0.0.13
 ip access-group 110 in
 ip nat inside
 duplex auto
 speed auto
!
interface Serial0/3/0
 ip address 10.0.0.14 255.255.255.252
 ip nat outside
 clock rate 2000000
!
router rip
!
ip classless
ip route 10.0.0.8 255.255.255.252 10.0.0.13 
ip route 192.168.11.0 255.255.255.128 10.0.0.13 
ip route 10.0.0.0 255.255.255.248 10.0.0.13 
ip route 192.168.10.0 255.255.255.128 10.0.0.13 
ip route 192.168.10.128 255.255.255.128 10.0.0.13 
ip route 200.0.0.0 255.255.255.248 10.0.0.13 
ip route 192.168.11.192 255.255.255.248 10.0.0.13 
!
access-list 1 permit 192.168.11.128 0.0.0.63
access-list 110 deny icmp 192.168.11.128 0.0.0.63 192.168.11.192 0.0.0.7
access-list 110 permit ip any any
!
end
```

### ISP Router
```text
!
version 15.1
hostname ISP
!
interface Serial0/3/0
 ip address 200.0.0.2 255.255.255.248
 clock rate 2000000
!
ip classless
!
end
```

### Engineer Switches

**Switch Destination Eng**
```text
!
hostname Switch_denstination_eng
!
interface FastEthernet0/1
 switchport mode trunk
!
interface FastEthernet0/2
 switchport access vlan 10
 switchport mode access
!
! (Interfaces Fa0/2 to Fa0/7 are in VLAN 10)
! (Interfaces Fa0/8 to Fa0/20 are in VLAN 20)
!
interface FastEthernet0/8
 switchport access vlan 20
 switchport mode access
!
end
```

**Switch Core Eng**
```text
!
hostname Switch_core_eng
!
interface FastEthernet0/1
 switchport mode trunk
!
interface FastEthernet0/2
 switchport mode trunk
!
interface FastEthernet0/3
 switchport mode trunk
!
end
```

**Switch Access Eng**
```text
!
hostname Switch_acess_eng
!
interface FastEthernet0/1
 switchport trunk allowed vlan 30
 switchport mode trunk
!
interface FastEthernet0/2
 switchport access vlan 30
 switchport mode access
!
interface FastEthernet0/3
 switchport access vlan 30
 switchport mode access
!
end
```

*(Note: Other intermediate switches like Switch3, Switch4, Switch5, and Switch6 are basic L2 pass-through devices with Fa0/1 and Fa0/2 configured as `switchport mode trunk` without any specific VLAN access assignment).*
