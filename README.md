# Linux-Based Software Router in GNS3 using Slackware 15

This project is a fully functional software router built from scratch using **Slackware 15**. It demonstrates how to turn a minimalist Linux machine into a router that provides internet access to an isolated local network (Kali Linux & VPCS nodes) inside a **GNS3** simulated environment. 

---

##  Tech Stack & Core Concepts

* **Network Simulator:** GNS3 
* **Virtualization:** Oracle VM VirtualBox
* **Router OS:** Slackware 15.0 (No GUI, Terminal only)
* **Client OS:** Kali Linux & VPCS
* **Key Concepts Used:** OSI Layer 3 Routing, IP Forwarding, NAT/Masquerade, `iptables` Firewall, DHCP Client.

---

<img width="940" height="619" alt="image" src="https://github.com/user-attachments/assets/ff8f09e2-cdbd-4446-9953-b634233e1169" />

##  Step-by-Step Configuration Guide

Here is exactly how I configured the Slackware router and the Kali Linux client:

### 1. Connecting the Router to the Internet (WAN)
To give the router internet access through the GNS3 Cloud (linked to the host's Wi-Fi), I ran the DHCP client to get an automatic IP:

```bash
dhcpcd eth0
```
Why? This binds eth0 to the external network gateway, fetching an active IP and DNS configurations dynamically from the physical environment.

2. Setting up the Local Network (LAN)
Next, I assigned a static IP to the interface connected to the internal switch. This IP will act as the Gateway for all local computers:
```bash
ifconfig eth1 192.168.1.1 netmask 255.255.255.0 up
```

Why? This defines 192.168.1.1 as the Default Gateway for the entire local subnet (192.168.1.0/24).

3. Turning on IP Forwarding
By default, Linux drops network packets that aren't meant for it. To make it act like a real router, I had to enable IP Forwarding in the kernel:
```bash
# Make the native routing script executable
chmod +x /etc/rc.d/rc.ip_forward

# Start the routing engine immediately in memory
/etc/rc.d/rc.ip_forward start
```

What it does: It allows the router to take packets from the LAN (eth1) and pass them along to the WAN (eth0).

4. Setting up NAT (Network Address Translation)
Local IP addresses (like 192.168.1.4) cannot communicate directly on the public internet. I used iptables to hide the local traffic behind the router's public IP:
```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

What it does: Right before a packet leaves the router, its private source IP is masked. When the internet replies, the router knows exactly which local PC to forward the response to.

<img width="940" height="728" alt="image" src="https://github.com/user-attachments/assets/c224d177-699d-40dc-bbe6-8c878aea5e14" />



5. Configuring the Kali Linux Client
   
Finally, I had to set up the Kali machine so it could talk to the new router:
```bash
# Assign a unique static IP address in the local pool
ip addr add 192.168.1.4/24 dev eth0

# Map all unknown target traffic to the Slackware Gateway
ip route add default via 192.168.1.1

# Bring the interface up
ip link set eth0 up
```
<img width="940" height="1167" alt="image" src="https://github.com/user-attachments/assets/069333ef-fa01-48e4-88ca-3c0e63903b78" />


### Troubleshooting & Verification
Setting up virtual networks always comes with bugs. Here are the main issues I solved during the project:

  * VirtualBox Adapter Conflicts: * Error: GNS3 couldn't link the NAT node due to missing vmnet8 interfaces.

Fix: Switched to the GNS3 Cloud node and bridged it directly to my host machine's active Wi-Fi adapter.

  * Wi-Fi Bridge Constraints: * Error: Windows host Wi-Fi drivers dropped bridged ICMP (ping) packets coming out of GNS3.

Fix: Validated the setup by pinging the gateway internally instead (ping -c 4 192.168.1.1), proving the local network was fully functional.


### Conclusion
This project was a great hands-on way to understand how routers actually work under the hood. Configuring iptables and IP forwarding manually from a Linux terminal teaches you a lot more about networking than just clicking "Next" on a commercial router interface.
