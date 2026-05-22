# How Devices Actually Connect To WiFi

Most people think “connecting to WiFi” simply means entering a password and magically getting internet. In reality, an absurd amount of networking machinery spins up in the background within a few seconds.

A modern device joining a WiFi network involves:

1. Radio-level communication (802.11)
    
2. Authentication
    
3. IP address allocation via DHCP
    
4. Routing
    
5. DNS resolution
    
6. TCP/IP session establishment
    

And the beautiful part is this: the router does not care whether the client is running Linux, Windows, macOS, Android, or an RTOS on an MCU.

The reason is because networking standards operate above the operating system layer.

---

# Step 1: The Device Finds The WiFi Network

When your laptop or embedded device enables WiFi, the wireless chipset begins scanning radio channels for nearby Access Points (APs). The router continuously broadcasts small management frames called **Beacon Frames**.

These frames contain information like:

- SSID (network name)
    
- Supported protocols
    
- Security type (WPA2/WPA3)
    
- Channel number
    
- Timing information

The client listens for these beacons and builds a list of visible networks.  At this stage, absolutely no IP traffic exists yet.

The device merely knows:

> “There is an access point named `HomeWiFi` operating on Channel 6 using WPA2.”

---

# Step 2: Authentication and Association

After selecting the network, the client begins authentication.

For WPA2/WPA3 Personal networks, this usually involves:

- Password validation
    
- Cryptographic key exchange
    
- Session key generation
    

This process establishes encrypted communication between the client and router. The important detail here is that WiFi itself is fundamentally a **Layer 2 protocol** in the OSI model.

That means WiFi handles:

- Frames
    
- MAC addresses
    
- Radio transmission
    

But not IP addressing yet.

At this point:

- The device is authenticated
    
- The device is associated with the AP
    
- Encrypted wireless communication exists
    
- But the device still does NOT have an IP address
    

---

# Step 3: DHCP Begins

Now comes the part most people associate with “getting connected”. The device acts as a DHCP client. The router usually acts as the DHCP server.

DHCP stands for:

> Dynamic Host Configuration Protocol

Its job is to automatically provide network configuration to clients. Without DHCP, every device would need manual IP configuration. 

The DHCP exchange is usually a 4-step process called DORA:

---

# DHCP DORA Process

## Discover

The client broadcasts:

> “Is there any DHCP server on this network?”

Since the client has no IP yet, this packet uses:

- Source IP: `0.0.0.0`
    
- Destination IP: `255.255.255.255`
    

This is literally a network-wide broadcast.

---

## Offer

The router replies:

> “I can offer you IP address 192.168.1.25”

The response also includes:

- Subnet mask
    
- Gateway address
    
- DNS server
    
- Lease time
    

---

## Request

The client responds:

> “I would like to use 192.168.1.25”

---

## Acknowledge

The router confirms:

> “Approved. That IP is now yours for this lease duration.”

Now the client officially becomes a valid node on the network.

---

# What The Router Actually Maintains

Routers internally maintain several important tables.

## DHCP Lease Table

Maps:

```text
MAC Address -> Assigned IP
```

Example:

```text
B8:27:EB:12:34:56 -> 192.168.1.25
```

This allows the router to remember which device owns which IP.

---

## ARP Table

Once communication begins, devices need MAC addresses for local delivery.

ARP resolves:

```text
IP Address -> MAC Address
```

Example:

```text
192.168.1.25 -> B8:27:EB:12:34:56
```

This is required because Ethernet and WiFi fundamentally deliver packets using MAC addresses at Layer 2.

---

# Why Linux and Windows Both Work

This is where abstraction layers become incredibly important.

The router does NOT interact with:

- Windows networking APIs
    
- Linux socket APIs
    
- Kernel syscalls
    
- User-space networking stacks
    

The router only sees standardized network packets.

That is the key insight.

---

# Standards Make Everything Compatible

Networking works because every operating system implements the same standards:

|Layer|Protocol|
|---|---|
|Application|HTTP, DNS|
|Transport|TCP, UDP|
|Network|IP|
|Data Link|Ethernet / WiFi|
|Physical|Radio / Electrical|

A Linux machine and a Windows machine may expose completely different APIs to applications internally, but once packets leave the NIC, they conform to the exact same standards.

The router only cares about:

- 802.11 frames
    
- IP packets
    
- TCP/UDP headers
    
- MAC addresses
    

Not the operating system.

---

# The OS Networking Stack

Internally, Windows and Linux have very different implementations.

## Linux

Linux networking is deeply integrated into the kernel.

Applications use:

```c
socket()
bind()
connect()
send()
recv()
```

The Linux kernel TCP/IP stack then handles:

- Packet construction
    
- Routing
    
- Congestion control
    
- Retransmissions
    
- Fragmentation
    

The WiFi driver finally pushes frames to the wireless chipset.

---

## Windows

Windows uses the Winsock API.

Example:

```c
WSASocket()
connect()
send()
recv()
```

Internally, Windows has its own networking stack implementation.

But eventually, Windows ALSO generates standards-compliant packets.

So from the router’s perspective:

```text
Linux Packet == Windows Packet
```

as long as both follow protocol standards.

---

# Why Routers Don't Need OS-Specific Logic

Imagine if routers had to understand:

- Linux kernel internals
    
- Windows NT networking
    
- Android frameworks
    
- BSD sockets
    
- RTOS networking APIs
    

Networking would be impossible.

Instead, the internet works because protocols define strict packet formats.

A TCP SYN packet generated by Linux must look identical to one generated by Windows.

That universality is the entire foundation of networking.

---

# What Happens After DHCP

Once the client has:

- An IP address
    
- Gateway
    
- DNS server
    

it can begin normal communication.

Typical flow:

## DNS Query

The client asks:

> “What is the IP address of google.com?”

---

## TCP Handshake

The client initiates:

```text
SYN
SYN-ACK
ACK
```

to establish a TCP connection.

---

## HTTP/HTTPS Traffic

Actual application data begins flowing.

The router now routes packets between:

- Local network
    
- ISP
    
- Wider internet
    

using NAT and routing tables.

---

# Where Embedded Devices Fit Into This

An MCU with WiFi behaves exactly the same way conceptually.

Whether using:

- ESP32
    
- STM32 + ESP8266
    
- Raspberry Pi Pico W
    

the device still:

1. Scans for APs
    
2. Authenticates
    
3. Performs DHCP
    
4. Receives IP configuration
    
5. Opens sockets
    
6. Sends TCP/UDP traffic
    

The difference is only implementation complexity.

An embedded stack might use:

- lwIP
    
- FreeRTOS+TCP
    
- Zephyr networking
    
- Vendor SDK stacks
    

But the generated packets still follow internet standards.

That is why your phone, laptop, smart bulb, and industrial controller can all coexist on the same router without issue.

---

# One Important Misconception

People often think:

> “The router gives internet.”

Not exactly.

The router mainly does three things:

1. Acts as an Access Point
    
2. Routes packets
    
3. Performs NAT between local and external networks
    

The actual internet connectivity comes from the WAN side connected to the ISP.

Without the ISP link, devices could still communicate locally over WiFi just fine.

---

# The Real Beauty Of Networking

The entire internet functions because every machine agrees on protocol standards.

Not operating systems.

Not programming languages.

Not hardware architectures.

A Linux server in Germany, a Windows laptop in India, and an ESP32 in a garage workshop can all communicate because they all speak the same protocol language:

```text
802.11
IP
TCP
UDP
DNS
HTTP
```

That standardization is one of the greatest engineering achievements ever built.