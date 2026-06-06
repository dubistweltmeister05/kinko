[[Twitter Posting]]
# How Devices Actually Connect To WiFi

Most people think connecting to WiFi means entering a password and magically getting internet access. In reality, an absurd amount of distributed networking machinery spins up underneath that single click. Within a few seconds, your machine performs radio scanning, 802.11 management exchanges, authentication, cryptographic key derivation, DHCP negotiation, ARP discovery, DNS resolution, TCP session establishment, NAT traversal, and often TLS encryption setup before a single webpage appears on the screen.

## **A Brief History of WiFi Standards**  
Before diving deeper, it helps to know that WiFi didn't start as the fast, secure protocol we use today. The original 802.11 standard from 1997 delivered only 2 Mbps. Subsequent amendments brought 802.11b (11 Mbps), 802.11a/g (54 Mbps), and eventually 802.11n (600 Mbps), 802.11ac (several Gbps), and 802.11ax (WiFi 6/6E). Each generation improved modulation schemes (from Barker coding to OFDMA), added MIMO antennas, and introduced better security. Today's devices still fall back to older rates for compatibility, which is why a 2024 laptop can connect to a 2008 router.

The most fascinating part is that the router fundamentally does not care whether the client is running Linux, Windows, macOS, Android, Zephyr, FreeRTOS, or a tiny lwIP stack running on an MCU. Once packets leave the network interface controller, every machine on Earth is forced into the same protocol grammar. That shared protocol language is what truly makes the internet interoperable.

## **Why "Interoperable" Is Harder Than It Sounds**  
Interoperability isn't automatic. Hundreds of engineers at the IEEE, IETF, WiFi Alliance, and vendor test labs spend years writing conformance tests. A router from TP-Link must work with a laptop from Apple, a phone from Samsung, and a smart plug from Tuya. The WiFi Alliance runs certification programs to ensure devices don't just follow the spec on paper but actually behave correctly in real-world radio environments with interference, fading, and different antenna designs. Without this invisible testing infrastructure, your laptop might only connect to routers from the same brand.

WiFi itself is not "the internet." That distinction matters enormously. WiFi is merely a wireless Layer 2 transport defined by the IEEE 802.11 specification. It defines radio modulation, airtime arbitration, MAC addressing, frame structures, retransmission behavior, and management signaling. It does not define IP addressing, routing, DNS, TCP, HTTP, or internet access. Those belong to higher layers of the networking stack.

## **The Difference Between a Router, a Switch, and an Access Point**  
In professional networking, these are separate boxes. A switch connects devices at Layer 2 within the same subnet. A router forwards packets between different subnets or to the internet. An access point converts wired Ethernet into wireless 802.11. Home "routers" combine all three plus a firewall and DHCP server. Understanding this distinction helps explain why you can have multiple WiFi access points (same SSID) but still need a router to leave your local network.

A modern consumer router is actually multiple independent networking systems hidden behind a single plastic enclosure. Inside that device exists an Access Point implementing 802.11 wireless communication, a Layer 2 switch handling local forwarding, a Layer 3 router forwarding IP packets between networks, a DHCP server assigning addresses, a NAT engine translating private and public traffic, a DNS forwarder caching queries, and usually a firewall enforcing packet filtering policies. Consumer networking hardware compresses decades of networking engineering into a tiny silent box sitting in the corner of a room.

## Connecting to the network

The entire process begins at the radio layer. When a WiFi chipset powers on, the NIC enters scanning mode and begins sweeping across RF channels listening for nearby Access Points. Routers continuously transmit Beacon Frames, which are specialized 802.11 management frames broadcast periodically, typically every 100 milliseconds. These frames advertise the existence and capabilities of the network. They contain metadata such as the SSID, supported PHY standards, encryption capabilities, supported data rates, channel width, timing synchronization parameters, and power management information.

At this stage, no IP traffic exists whatsoever. No sockets exist. No routing exists. No TCP sessions exist. The client merely learns that an Access Point exists nearby and what capabilities it supports. The entire interaction is still purely Layer 2 radio communication.

Modern clients usually perform both passive and active scanning. In passive scanning, the device simply listens for beacon frames and builds a list of visible networks. In active scanning, the client transmits Probe Requests asking whether a specific SSID exists nearby. Access Points matching that SSID respond with Probe Responses. Active scanning dramatically reduces discovery latency, which is why modern devices can populate a WiFi list almost instantly.

Once a network is selected, the client begins the authentication and association process. Historically, 802.11 authentication was nearly meaningless. Early implementations effectively consisted of the client saying "authenticate me" and the AP replying "okay." Modern WiFi security instead relies on WPA2 or WPA3 layered on top of the basic 802.11 authentication framework.

The famous WPA2 four-way handshake is where real cryptographic security begins. Contrary to popular belief, the WiFi password itself is not directly used as the encryption key for traffic. Instead, the password derives a Pairwise Master Key, after which both the client and AP exchange nonces and derive temporary session keys. This process establishes encryption keys for unicast and multicast traffic while simultaneously protecting against replay attacks. The resulting traffic encryption is usually AES-CCMP.

At this point, encrypted wireless communication exists between the client and the AP, but the device still does not possess an IP address. It exists only as a Layer 2 participant identified by its MAC address.

The association phase formally registers the client with the Access Point. The AP now maintains state information for the device, including encryption keys, QoS capabilities, sequence counters, supported transmission rates, and power-saving state. The client officially becomes part of the Basic Service Set, which is the logical collection of wireless stations managed by the AP.

Only after this does Layer 3 networking begin.

## Getting access to the internet via said network

The device now needs an IP address, subnet mask, default gateway, and DNS server information. This is where DHCP enters the picture. DHCP itself operates over UDP, which surprises many people because the client does not yet possess an IP address. The solution is broadcasting. The client transmits packets using source IP `0.0.0.0` and destination IP `255.255.255.255`, effectively shouting into the local network asking whether any DHCP server is available.

This process is commonly called DORA: Discover, Offer, Request, Acknowledge.

The client first broadcasts a Discover packet asking whether any DHCP server exists on the network. The router responds with an Offer packet proposing an available IP address along with additional configuration parameters such as subnet mask, lease duration, DNS servers, and gateway information. The client then sends a Request packet indicating it wishes to use that offered address, after which the router replies with an Acknowledge packet officially leasing the address to the client.

At that exact moment, the device becomes a valid Layer 3 node on the network.

Internally, the router maintains several critical tables that make this communication possible. The DHCP lease table maps MAC addresses to assigned IP addresses, allowing the router to remember which device owns which address. Simultaneously, ARP tables maintain mappings between IP addresses and MAC addresses because Ethernet and WiFi fundamentally deliver frames using Layer 2 addressing, not IP addresses directly.

This distinction is extremely important. IP addresses are logical identifiers used for routing between networks, while MAC addresses are hardware identifiers used for local delivery inside a broadcast domain. When a device wishes to communicate locally, it must first resolve the destination IP into a MAC address using ARP. Only then can the frame actually be transmitted over Ethernet or WiFi.

What makes networking extraordinary is that none of this machinery depends on the operating system running on the client device. The router does not understand Linux kernel internals, Windows NT networking APIs, BSD socket layers, or embedded RTOS abstractions. It only understands standardized packet formats.

Internally, Linux and Windows networking stacks are radically different systems. Linux networking is deeply integrated into the kernel and exposes socket interfaces like `socket()`, `bind()`, `connect()`, `send()`, and `recv()`. The Linux kernel TCP/IP stack handles routing decisions, congestion control, packet retransmissions, fragmentation, checksum generation, and traffic scheduling before eventually passing frames to the NIC driver.

Windows uses the Winsock API instead, exposing interfaces like `WSASocket()` and its own internal networking architecture. Yet despite these differences, both systems ultimately emit standards-compliant Ethernet, IP, TCP, UDP, and 802.11 frames.

From the router's perspective, a TCP SYN packet generated by Linux must look identical to one generated by Windows. That universality is the entire foundation of networking interoperability.

Once DHCP completes, normal communication can finally begin. Usually the first operation is DNS resolution. The client asks a DNS server to resolve a human-readable hostname like `google.com` into an IP address. After receiving the destination IP, the client initiates a TCP three-way handshake consisting of SYN, SYN-ACK, and ACK packets. Only after this stateful transport session is established does actual application-layer traffic begin flowing.

## Letting the packets flow

For HTTPS traffic, the process continues even further with TLS negotiation, certificate validation, cipher suite agreement, ephemeral key exchange, and encrypted session establishment before application payloads are exchanged.

Meanwhile, the router performs Network Address Translation because devices inside home networks typically use private RFC1918 addresses that are not globally routable on the public internet. NAT rewrites packet headers, tracks connection state, and maps internal private addresses onto a shared public IP provided by the ISP. Without NAT, home routers would require globally routable addresses for every local device.

Embedded systems fundamentally follow this exact same process. An ESP32, STM32 paired with an ESP8266, Raspberry Pi Pico W, industrial PLC, smart bulb, or automotive telematics module all perform the same conceptual workflow. They scan for Access Points, authenticate, associate, negotiate DHCP leases, resolve DNS, open sockets, and transmit TCP or UDP traffic. The only thing that changes is implementation complexity.

An embedded networking stack may use lwIP, FreeRTOS+TCP, Zephyr networking, or vendor SDK implementations instead of full Linux networking subsystems, but the generated packets still conform to the same standards. That is why an MCU with a few hundred kilobytes of RAM can coexist perfectly on the same network as enterprise Linux servers and gaming desktops.

The internet works because every machine agrees to speak the same protocol language. Not the same operating system. Not the same programming language. Not the same CPU architecture. A Linux server in Germany, a Windows laptop in India, and an ESP32 in a garage workshop can all communicate flawlessly because every device eventually reduces itself to the same universally agreed packet structures:

```text
802.11
Ethernet
IP
TCP
UDP
DNS
HTTP
TLS
```

That standardization is arguably one of the greatest engineering achievements humanity has ever built.

**What Actually Happens When WiFi "Drops" or Is "Slow"**  
Most connectivity problems happen not at the password or IP layer but during radio contention or interference. When many devices share the same channel, 802.11's CSMA/CA protocol forces them to wait and back off. Slow WiFi often means high airtime utilization, not a bad internet connection. Likewise, "connected but no internet" usually means DHCP succeeded but the default gateway or DNS forwarder failed—Layer 2 works, but Layer 3 is broken. The next time WiFi feels slow, check whether the issue is local (high latency to the router) or beyond (high latency to 8.8.8.8).

## Conclusion
## Conclusion

What makes WiFi fascinating is not the radio waves, the encryption, or even the routing itself. It is the fact that billions of completely different machines can all participate in the same conversation.

A Linux server running on x86 hardware, a Windows laptop, an Android phone, an ESP32 running FreeRTOS, and an industrial PLC built fifteen years ago all speak through the exact same protocol layers. Beneath radically different kernels, drivers, APIs, and hardware architectures, every device eventually reduces itself into the same sequence of standardized frames and packets. That universality is what makes networking work.

The moment a device connects to WiFi, an enormous cascade of systems activates underneath the surface: RF negotiation, authentication, encryption, DHCP leasing, ARP discovery, DNS resolution, TCP state synchronization, NAT traversal, and often TLS cryptography. All of this happens in seconds, silently, thousands of times per day, across billions of devices.

Most users never see any of it. They simply click a network name and expect the internet to appear. But underneath that single click exists decades of engineering, protocol design, mathematical rigor, hardware interoperability testing, and global standardization work coordinated across vendors, operating systems, silicon manufacturers, and networking organizations.

The internet does not function because every machine is the same.

It functions because every machine agrees on the same language.

And honestly, that may be one of the greatest examples of large-scale engineering cooperation humanity has ever built