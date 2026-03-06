# Hybrid Enterprise Network Lab & Bare-Metal Linux Gateway

## Project Overview
This project demonstrates the design and deployment of a custom hybrid network lab. To bypass the limitations of consumer-grade hardware, a Ubuntu server machine was used to act as a primary NAT gateway and edge router. This setup bridges a physically isolated local area network (LAN) to a wide area network (WAN) while allowing for deep packet inspection and custom routing rules.

## Hardware & Infrastructure
* **Virtualization Host / Client:** Dell G15 (i7, 32GB RAM , RTX 3060)
* **Bare-Metal Gateway / Router:** Ubuntu Linux Machine  (i5-8250U , 16GB RAM)
* **Layer 2 Switch:** Huawei WS5200 V3 (Demoted from gateway duties to act purely as a switch)(cannot flash custom firmware)

## Network Topology
1. **WAN Uplink:** The Ubuntu Gateway connects to the internet via its wireless interface (`<wifi-interface>`)(wlp2s0).
2. **LAN Downlink:** The Ubuntu Gateway connects to the Huawei switch via its physical Ethernet interface (`<ethernet-interface>`)(elp2s0).
3. **Client Connection:** The Dell G15 connects to the Huawei switch, relying entirely on the Ubuntu Gateway for DHCP, DNS routing, and NAT to reach the outside world.

## Execution & Configuration Steps

### 1. Interface Provisioning
NetworkManager (`nmcli`) was utilized to establish the physical links.
* Connected the wireless interface to the external WAN.
* Assigned a static IP address to the Ethernet interface to serve as the default gateway for the isolated lab network(only router an ubuntu server have static IP's).

### 2. Kernel Manipulation (IP Forwarding)
By default, the Linux kernel drops packets not destined for itself. IPv4 forwarding was enabled to allow the kernel to pass traffic between the physical Ethernet port and the wireless card.
```bash
# Temporary enablement for testing
sudo sysctl -w net.ipv4.ip_forward=1
```

3. Firewall Rules & NAT (Masquerading)
Traffic originating from the internal lab network utilizes a private, custom IP scheme. To allow this traffic to pass through the home router and onto the internet, iptables was configured to masquerade the outbound packets, translating their source IPs to the Ubuntu machine's WAN IP.

```Bash
# Append NAT masquerade rule to the POSTROUTING chain
sudo iptables -t nat -A POSTROUTING -o <wifi-interface> -j MASQUERADE
```
4. Infrastructure Persistence (Hardening)
To ensure the gateway survives reboots and power cycles, the runtime configurations were made permanent.

Kernel Persistence: Modified /etc/sysctl.conf and uncommented net.ipv4.ip_forward=1.

Firewall Persistence: Installed iptables-persistent (or netfilter-persistent) to automatically save and restore the NAT masquerade rules on boot.

```Bash
# Save current iptables rules to persistence file
sudo netfilter-persistent save
```
Interface Persistence: Configured nmcli connection profiles to autoconnect yes upon system startup.

Results
The system successfully routes ICMP and TCP/UDP traffic from the isolated Dell host, through the Huawei switch, across the Ubuntu kernel, translated via NAT, and out to the public internet. The environment is now fully prepared for advanced traffic interception and protocol analysis.

