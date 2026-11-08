[[OpenWRT - Custom Network Wide Ad-Blocker]]
## Project brief



**Project goal:** Build a capable OpenWRT-based home router that provides network-wide ad and tracker filtering, while using the project as a practical lab for embedded Linux, networking, Wi-Fi drivers, and firmware development.

**Core idea:** Put an OpenWRT-capable router between the existing ISP router/ONT and the devices you want to manage. Configure it as the network gateway (or, where appropriate, as an access point), run DNS-based filtering, and progressively explore the Linux networking and wireless stack.

This is intended to be more than a one-time ad-blocking installation: the long-term goal is a reproducible, custom OpenWRT image and a deeper
understanding of the hardware/software boundary in Wi-Fi systems.

------------------------------------------------------------------------

## 1. Target architecture

A simple gateway arrangement looks like this:

``` text
Internet
   |
ISP router / ONT
   |
   | Ethernet
   v
OpenWRT router
   |-- DNS filtering (AdGuard Home or OpenWRT adblock)
   |-- DNS resolver / DHCP
   |-- Firewall (nftables / fw4)
   |-- Wi-Fi access point (hostapd)
   |-- Monitoring and logging
   |
   +---- Laptop
   +---- Phone
   +---- TV / console
   +---- IoT devices
```

Devices use the OpenWRT router's DNS resolver. When a device asks to resolve a domain, the resolver checks it against configured blocklists. If the domain is blocked, the resolver returns a blocked response instead of resolving it normally.

- Blocking Scripts
- IP Allow-lists
- Usage Telemetry Scripts that can be accessed on a static IP
- Something else that I can come up with
### What this can and cannot block

DNS filtering can block many advertising, tracking, telemetry, and malicious domains across devices without installing an extension on each device. 
It is not a universal ad blocker:

-   Ads served from the same domains as legitimate content are difficult to block selectively.
-   YouTube and many in-app advertisements are not reliably blocked by DNS filtering.
-   Devices and applications can use their own encrypted DNS or hard-coded DNS servers unless network policy addresses those paths.
-   Blocking lists can cause false positives; allowlists and per-client rules are useful.

------------------------------------------------------------------------

## 2. Software choices

### AdGuard Home

A strong starting point for a home deployment.

-   DNS-based ad and tracker filtering.
-   Web dashboard with query statistics.
-   Per-client configuration and custom rules.
-   Supports encrypted upstream DNS options.
-   Useful visibility into which clients are making DNS requests.

**Suggested role:** Start with AdGuard Home to get a working result and learn DNS filtering operations.

### OpenWRT `adblock`

A more OpenWRT-native approach.

-   Integrates with OpenWRT's resolver ecosystem.
-   Lightweight compared with running a full separate dashboard service.
-   Good for learning OpenWRT packages, service configuration, and
    resolver integration.

**Suggested role:** Explore after the first working setup, or choose it if keeping the router lightweight is a priority.

### Pi-hole

A mature DNS sinkhole that can run on a separate Linux host, such as a Raspberry Pi or another always-on machine.

**Suggested role:** An alternative architecture if you want DNS filtering to run independently of the router. It is not necessary to buy a separate SBC just to get started.

### Recommended progression

1.  Start with AdGuard Home for visibility and ease of use.
2.  Learn how OpenWRT distributes DNS settings through DHCP.
3.  Explore OpenWRT's native ad-blocking options.
4.  As a later exercise, write a small DNS filtering/forwarding service and package it for OpenWRT.

------------------------------------------------------------------------

## 3. OpenWRT and Wi-Fi architecture

OpenWRT is a Linux distribution designed for network devices. It provides a useful environment for learning:

-   Linux kernel and device drivers.
-   Routing, bridging, interfaces, and IP addressing.
-   DHCP and DNS.
-   Firewalling with nftables and OpenWRT's `fw4`.
-   Wireless configuration through hostapd.
-   Package management and cross-compilation.
-   System services, logs, and configuration management.

A simplified view of the Wi-Fi software/hardware stack:

``` text
User-space applications and tools
  LuCI, hostapd, wpa_supplicant, iw
                  |
Linux wireless subsystem
  cfg80211 / nl80211 / mac80211
                  |
Hardware-specific kernel driver
  e.g. mt76, ath11k, brcmfmac
                  |
Chipset firmware (where applicable)
                  |
Wi-Fi MAC/PHY hardware and RF front-end
                  |
Antennas
```

The exact architecture varies by chipset and driver.

### Important distinction: OpenWRT firmware vs Wi-Fi chipset firmware

These are related but different areas of work.

-   **OpenWRT firmware:** The Linux-based operating system image installed on the router. You can configure it, build it, add packages, modify kernel configuration, and work on supported drivers. 
-   **Wi-Fi driver:** The host-side Linux kernel code that communicates with the wireless hardware.
-   **Chipset firmware:** Low-level code that may execute on processors inside the Wi-Fi chipset. It is often vendor-provided and may not have publicly available source code.

You can learn a great deal about wireless networking and Linux drivers without having the source code for the chipset's internal firmware.

### Chipset families worth investigating

-   **MediaTek `mt76`:** A substantial open-source Linux wireless driver family and an interesting target for hands-on exploration.
-   **Qualcomm `ath11k`:** An important driver family for supported Qualcomm Wi-Fi 6 hardware.
-   **Broadcom:** Support varies by device and may depend on proprietary components; investigate the exact model carefully.

Do not choose a router based only on the brand or advertised Wi-Fi standard. The exact hardware revision, chipset, flash layout, RAM, and upstream driver support matter.

------------------------------------------------------------------------

## 4. Hardware selection in India

### Suggested hardware requirements

  Component   Practical target
  ----------- ------------------------------------------------
  RAM         256 MB as a baseline; 512 MB or more preferred
  Flash       128 MB preferred where possible
  CPU         Supported ARM or MIPS platform
  Ethernet    Gigabit Ethernet
  Wi-Fi       Chipset with mature OpenWRT support
  Storage     USB or eMMC is a useful bonus

These are selection guidelines, not universal minimum requirements. The services you intend to run determine how much memory and storage you
need.

### Models to investigate

These are candidates to research, not unconditional purchase recommendations. Verify the exact hardware revision and current upstream OpenWRT support before buying.

#### GL.iNet GL-MT3000 (Beryl AX)

-   MediaTek MT7981B platform.
-   512 MB RAM.
-   Wi-Fi 6.
-   Compact form factor.
-   Worth investigating as a capable first device.

Check whether the exact model/revision has the upstream OpenWRT support and installation path you want. GL.iNet's vendor firmware is OpenWRT-based, but it is not necessarily identical to a clean upstream OpenWRT installation.

#### GL.iNet GL-MT6000 (Flint 2)

-   MediaTek MT7986AV platform.
-   1 GB RAM.
-   Wi-Fi 6.
-   More memory headroom for multiple services and experiments.

Again, verify exact upstream support, installation procedure, and recovery options before purchase.

### Where to verify compatibility

Use the official OpenWRT resources:

-   Table of Hardware: https://openwrt.org/toh/start
-   Supported devices: https://openwrt.org/supported_devices
-   Firmware installation documentation:
    https://openwrt.org/docs/guide-user/installation/start

Check all of the following before ordering:

1.  Exact model name and hardware revision.
2.  Exact chipset and Wi-Fi driver.
3.  Whether the device is supported by current stable OpenWRT.
4.  Whether support is for the exact revision, not merely a similarly named model.
5.  Flash and RAM capacity.
6.  Installation method and whether it requires vendor firmware first.
7.  Recovery procedure if flashing fails.
8.  Whether Wi-Fi functionality is fully supported or has known limitations.

### Using the existing ISP router

You do not necessarily need to replace the ISP-provided ONT/router.

A straightforward arrangement is:

``` text
ISP router LAN  --->  OpenWRT WAN  --->  Your devices
```

You can initially leave the ISP router in place and connect OpenWRT behind it. This is convenient when the ISP controls the fiber ONT or does not allow third-party ONTs.

Considerations:

-   This arrangement can create double NAT.
-   Double NAT is often acceptable for ordinary browsing and DNS filtering.
-   It can complicate inbound connections, port forwarding, some VPN setups, and hosting services.
-   A bridge mode or a carefully configured access-point arrangement may be preferable later, depending on the ISP equipment and requirements.

------------------------------------------------------------------------

## 5. Indicative project budget

These are rough planning estimates, not verified live Indian retail
prices.

  Item                            Approximate budget
  ----------------------------- --------------------
  Capable OpenWRT router              ₹4,000--10,000
  Higher-end router                  ₹8,000--15,000+
  Ethernet cable                           ₹200--500
  USB-UART adapter (optional)              ₹200--400
  USB storage (optional)                 ₹500--1,000

A sensible initial budget is approximately **₹6,000--10,000**, depending on the router selected and local availability.

You do not need a Raspberry Pi or separate server to get started. If you later want to run heavier services, additional storage, or independent network monitoring, you can expand the setup. 

------------------------------------------------------------------------

## 6. Development roadmap

The project is divided into stages so that each one produces a working result.

### Stage 1 --- Get OpenWRT running

**Tasks**

-   Select a supported router and verify its exact hardware revision.
-   Record the stock firmware version and recovery procedure.
-   Install upstream OpenWRT, following the device-specific instructions.
-   Learn SSH access, LuCI, UCI, package management, and system services.
-   Inspect network interfaces, bridges, routes, and firewall zones.
-   Learn how WAN, LAN, DHCP, and wireless configuration fit together.

**Deliverable:** A working OpenWRT router with a documented recovery path.

### Stage 2 --- Build network-wide DNS filtering

**Tasks**

-   Install and configure AdGuard Home.
-   Configure DHCP to distribute the intended DNS resolver to clients.
-   Configure upstream DNS.
-   Add reputable blocklists.
-   Create custom allowlists and denylists.
-   Inspect query logs and per-client statistics.
-   Test phones, laptops, TVs, and other clients.

**Deliverable:** Network-wide ad and tracker filtering with visibility
into DNS activity.

### Stage 3 --- Understand network enforcement

**Tasks**

-   Learn OpenWRT firewall zones and nftables (`fw4`).
-   Understand how DNS requests travel through the network.
-   Experiment with redirecting ordinary DNS traffic to the chosen resolver.
-   Test what happens when a client tries to use a different DNS server over port 53.
-   Investigate DNS-over-HTTPS and DNS-over-TLS bypasses.
-   Account for IPv6, not just IPv4.
-   Learn the trade-offs between enforcement, compatibility, and privacy.

**Deliverable:** A documented network DNS policy, including known bypasses and limitations.

### Stage 4 --- Explore wireless internals

**Tasks**

-   Inspect hostapd configuration and logs.
-   Explore `iw`, `iwinfo` where available, and wireless interface state.
-   Study cfg80211, nl80211, and mac80211.
-   Read the source for the device's actual wireless driver.
-   Learn AP and station modes.
-   Experiment with channel selection and transmit power within local regulatory limits.
-   Investigate roaming and 802.11k/v/r where hardware and client support allow it.
-   Capture and interpret useful kernel logs.
 
**Deliverable:** A practical understanding of the wireless stack and the selected router's driver.

### Stage 5 --- Build a custom OpenWRT image

**Tasks**

-   Set up the OpenWRT build environment.
-   Build an image for the exact target device.
-   Learn package selection and image configuration.
-   Cross-compile a small userspace program.
-   Add a custom package or service.
-   Modify kernel configuration where appropriate.
-   Build reproducible firmware images.
-   Test upgrades and recovery safely.

**Deliverable:** A reproducible custom OpenWRT image with your own package or service.

------------------------------------------------------------------------

## 7. Further experiments

  -----------------------------------------------------------------------
  Experiment                          Concepts to learn
  ----------------------------------- -----------------------------------
  DNS query logging                   DNS resolution, caching, client
                                      identification

  DNS interception                    nftables, packet redirection,
                                      firewall policy

  Per-device filtering                DHCP leases, client identity,
                                      resolver policy

  Guest Wi-Fi                         Network isolation and firewall
                                      zones

  IoT VLAN                            VLAN tagging, bridges, inter-VLAN
                                      routing

  Wi-Fi roaming                       802.11k/v/r and client behaviour

  Traffic monitoring                  Linux networking, counters, packet
                                      inspection

  Custom DNS service                  C programming, sockets, event
                                      loops, Linux services

  Custom OpenWRT package              Build system, package metadata,
                                      init scripts

  Driver investigation                Kernel source, logs, wireless
                                      subsystem APIs
  -----------------------------------------------------------------------

### A particularly worthwhile personal project: a DNS filtering daemon in C

Once the router is working, implement a small DNS filtering service as a
learning exercise.

Possible components:

1.  DNS request listener.
2.  Configurable domain blocklist.
3.  DNS cache.
4.  Upstream DNS forwarding.
5.  Query logging and statistics.
6.  Configuration file and reload mechanism.
7.  OpenWRT init script and package integration.

Start with a narrow, testable subset of DNS functionality. DNS has many record types, response rules, caching behaviours, and edge cases; a learning daemon should not be treated as a production-grade resolver until it handles them correctly.

**Deliverable:** A small C-based service that you can build, deploy, debug, and eventually package for OpenWRT.

------------------------------------------------------------------------

## 8. Project principles and safety notes

-   Verify device support before purchasing.
-   Keep a copy of the original firmware and device-specific recovery instructions.
-   Do not experiment with firmware flashing on the only router needed for internet access.
-   Prefer a separate test network while learning firewall and routing changes.
-   Back up OpenWRT configuration before major changes.
-   Keep track of the exact firmware build and package versions.
-   Be careful with DNS interception: it can break applications that expect a particular resolver.
-   Configure IPv6 deliberately; otherwise, clients may bypass IPv4-only filtering.
-   Treat blocklists as policy inputs, not unquestionable truth. Monitor false positives.
-   Follow the applicable Indian wireless regulatory requirements; do not experiment with unauthorized transmit power or frequencies.

------------------------------------------------------------------------

## 9. Initial decisions to make

Before buying hardware, settle these three points:

-   **Budget:** Under ₹5,000; ₹5,000--8,000; ₹8,000--12,000; or
    ₹12,000--20,000.
-   **Primary objective:** Practical ad-blocking router; both practical
    router and deep firmware learning; or primarily Wi-Fi
    driver/firmware development.
-   **ISP setup:** JioFiber, Airtel Xstream Fiber, local fiber ISP, or
    another connection.

Use those answers to shortlist devices based on upstream OpenWRT support, chipset openness, RAM/flash, recovery options, and Indian availability.

------------------------------------------------------------------------

## 10. Useful official resources

-   OpenWRT documentation: https://openwrt.org/docs/start
-   OpenWRT Table of Hardware: https://openwrt.org/toh/start
-   OpenWRT supported devices: https://openwrt.org/supported_devices
-   OpenWRT build system:
    https://openwrt.org/docs/guide-developer/toolchain/buildsystem_essentials
-   OpenWRT firewall documentation:
    https://openwrt.org/docs/guide-user/firewall/firewall_configuration
-   Linux wireless documentation: https://wireless.docs.kernel.org/
-   AdGuard Home: https://github.com/AdguardTeam/AdGuardHome
-   Pi-hole documentation: https://docs.pi-hole.net/

------------------------------------------------------------------------

## Project success criteria

The first complete milestone is achieved when:

-   [ ] OpenWRT runs reliably on the selected router.
-   [ ] The ISP connection works through the OpenWRT device.
-   [ ] Client devices receive the intended DNS settings.
-   [ ] DNS filtering works for IPv4 and IPv6 clients.
-   [ ] Query logs identify clients and blocked domains.
-   [ ] DNS bypass behaviour has been tested and documented.
-   [ ] A configuration backup and firmware recovery procedure are available.

The longer-term milestone is achieved when a custom OpenWRT image builds reproducibly and includes at least one package or service developed as
part of this project.

**Guiding idea:** Build a useful home networking appliance first, then use it as a platform to learn the internals---rather than treating ad blocking as the end goal.
