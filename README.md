# Linux-Based Software Router in GNS3 using Slackware 15 
> A comprehensive guide to configuring a Linux virtual machine to act as a fully functional router with NAT, DHCP, and BIND DNS services.

This project is a fully functional software router built from scratch using **Slackware 15**. It demonstrates how to turn a minimalist Linux machine into a router that provides internet access to an isolated local network (Kali Linux & VPCS nodes) inside a **GNS3** simulated environment. 

---

##  Tech Stack & Core Concepts

* **Network Simulator:** GNS3 
* **Virtualization:** Oracle VM VirtualBox
* **Router OS:** Slackware 15.0 (No GUI, Terminal only)
* **Client OS:** Kali Linux & VPCS
* **Key Concepts Used:** OSI Layer 3 Routing, IP Forwarding, NAT/Masquerade, `iptables` Firewall, DHCP Client, DHCP Server, DNS Server.

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


# ====================PART 2================
I decided to add a DHCP server and DNS server on the router.

To provide IP addresses in the local network, we first had to configure a static IP address on the LAN interface (`eth1`), because I didn't save the previous configuration, and then start the DHCP server.

So, I entered the configuration file for the IP adresses and I manually configured the static IPv4 address for eth1 - 192.168.1.1
```bash
nano /etc/rc.d/rc.inet1.conf
```
<img width="409" height="461" alt="image" src="https://github.com/user-attachments/assets/5809e6b0-d0ed-4f32-b294-ab6fafffd2fe" />

Also, I modified the topology, replacing the cloud with NAT, because my pings to google.com didn't work. <img width="667" height="433" alt="image" src="https://github.com/user-attachments/assets/64e8a250-cc15-4bd6-ad70-3342cef12224" />

## DHCP Configuration
I modified the dhcp config file
```bash
nano /etc/dhcpd.conf
```
<img width="794" height="598" alt="image" src="https://github.com/user-attachments/assets/08a103e9-c582-44a5-840e-29c49642e0e0" />

Now, our DHCP Server will lease adresses ranging from 192.168.1.100 to 192.168.1.200, with the default gateway of 192.168.1.1 and the broadcast address of 192.168.1.255. The lease time is 600 seconds (=10 minutes), and the maximum time that a PC may use a certain IP address is 7200 seconds (=2 hours), after which it will have to Request a new IP address from the server.

Starting the DHCP service
```bash
chmod +x /etc/rc.d/rc.dhcpd
/etc/rc.d/rc.dhcpd start
```

Configuring DHCP on VPCS:
```bash
PC1> ip dhcp
```

Configuring DHCP on Kali Linux:
```bash
(root kali)# dhclient eth0
```

## DNS Configuration
```bash
nano /etc/named.conf
```
<img width="796" height="599" alt="image" src="https://github.com/user-attachments/assets/3b82baa7-fac3-4d37-9b6a-4e3017c84e73" />

Explanation:

listen-on port 53 { 127.0.0.1; 192.168.1.1; } -> I added this line to tell the DNS service exactly where it should be "listening" for incoming questions. I gave it 127.0.0.1 (the router itself) and 192.168.1.1 (the IP address of our router's internal LAN interface). Port 53 is just the standard port that DNS traffic uses.

allow-query: I added our specific subnet (192.168.1.0/24). This acts as a security rule so only our lab PCs can use this DNS server, blocking outside traffic.

forwarders: I put in public servers like 8.8.8.8 (Google's DNS) and 1.1.1.1 (Cloudflare's DNS). Since our local router doesn't know every IP on the internet, it asks these servers whenever it doesn't know the answer.

recursion yes;: This simply allows the router to go out, grab the answers from those forwarders, and bring them back to the client.

### Problem: I tried pinging google.com on a VPC. DNS works, but my pings are failing. Why is that? 

<img width="655" height="416" alt="image" src="https://github.com/user-attachments/assets/dcbf6362-db87-4512-9a70-d34cea465fcd" />

### Solution: NAT isn't working, because I didn't save what I did last time in the "rc.local" (equivalent to a ".startup-config") file.

<img width="797" height="599" alt="image" src="https://github.com/user-attachments/assets/5a1bae8f-011b-451b-a81c-cf7192c7ffc4" />

## Conclusion

DHCP is working.

<img width="1112" height="110" alt="image" src="https://github.com/user-attachments/assets/41f040d9-c28b-4a93-a66c-ef686b0ed9b3" />

<img width="623" height="479" alt="image" src="https://github.com/user-attachments/assets/0bd615d4-1203-4a8a-9b51-575605f92ad7" />


DNS is working.

<img width="521" height="115" alt="image" src="https://github.com/user-attachments/assets/c3c552ce-3db7-4137-95ea-0cd9c04c298d" />
<img width="735" height="181" alt="image" src="https://github.com/user-attachments/assets/96f903c7-37fe-44f7-936d-99413aa6da69" />




