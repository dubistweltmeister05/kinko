[[Networking]]

Right lads, I got a question for you lot. We all have used the hotspot of our phones, but I be that most of us are oblivious to the insane engineering that goes on at the back of all of this. 

Well, that ends today. Keep in mind, this discussion uses code and other examples from a Linux machine, because as everyone knows, you aren't a real engineer if you haven't convinced yourself that spending 10 hours trying to get the Wi-Fi on your laptop to work is fun!

And yes, this shall be a very unhinged, and unapologetic read. I will go off on tangents, but I promise you, I'll pull it back and bring the story together quite well.

With that being said, let's read on and see what exactly happens when we flick a button and share data with our friends; or if you don't have any, with other devices!

---

## 1. The Two Roles of Wi-Fi
Ladies and Gentlemen. A short view in the past (iykyk).....

A Wi-Fi interface onto any device is a complex bit of engineering. What we see as "wlan0" on our PCs when we type in ipconfig, is just the software label. At the depths of your machine, there lies an RF front-end, a complex configuration of antennas, mixers, filters and amplifiers, that handle the 2.4/5GHz radio waves, which we shall convert into a baseband signal - a basic, untouched signal that shall be modulated to communicate what we want to send over the medium. 

The baseband is fed into the physical layer of the Wi-Fi interface, which modulates/demodulates that baseband wave, in a multitude of possible ways - OFDM, QAM, etc, and performs other cleanup tasks, like error correction, synchronization, and MIMO (Multiple-In/Multiple-Out)operations.

Above this sits the MAC Layer, that manages the implementation of the IEEE 802.11 standard, which governs the framing of data, packet creation, retransmission, acknowledgements, and even encryption. These MAC/PHY blocks run under the control of a small **firmware image** that your Linux system loads into the Wi-Fi chip at runtime, ensuring time-critical tasks like beaconing or responding to probes happen within microseconds.

On the host CPU, a **kernel driver** (e.g. `ath10k`, `brcmfmac`, `iwlwifi`) communicates with the chip over PCIe, SDIO, or USB, exposing it as a Linux network device. This is where `wlan0` comes in: it’s the **netdev abstraction** created by the driver, which then integrates into the wider Linux networking stack. Tools like `wpa_supplicant` or `hostapd` configure it by speaking nl80211 (a netlink interface) to the kernel, which in turn instructs the Wi-Fi firmware. So, `wlan0` is just the tip of the iceberg—the visible Linux handle for a complex hardware-software system spanning antennas, radios, DSPs, embedded firmware, kernel drivers, and the full TCP/IP stack.

Now, every Wi-Fi interface can act in 2 modes. It can either connect to a wireless network, and provide connectivity for the device that it is linked it - STA(Client) mode. Or, it can share the connection from a network that is is connected to via a physical wire - AP(Access Point) mode (there are interfaces that can connect to a wireless network and share the same network's access to others at the same time, but we aren't talking about those rn.) 

These two modes are managed by different tools in the :

- **wpa_supplicant**:
    - This is the user-space daemon that is used to handle the wlan0 interface in the client mode. 
    - It handles scanning, authentication, WPA2/3 handshakes, and maintaining the link with an upstream AP.        
    - Essentially, it’s your Wi-Fi “driver assistant.”
    - Here is what it actually does - 
	    - First, it'll talk to the kernel to say wassup, and then get the control of the wlan0 interface via the appropriate driver. 
	    - Then, it'll handle the scanning of the SSIDs, parsing through all the responses and beacons to try and find the SSID that we have configured it to connect to (via a conf file).
	    - Once it gets the intended SSID - it'll initiate the authentication process. Basically, it'll enter the password, but on it's own now (just kidding, we have to specify the password our self, in the conf file). 
	    - As that goes on, there is a 4-way handshake that goes on in the back, to generate a PTK/GTK encryption key with the AP. Once all is set, the keys are loaded into the driver/firmware. 
- **hostapd (Host Access Point Daemon)**:  
    - This, turns the device into an Access Point.       
    - It configured the WiFi card to keep broadcasting beacon frames into the interwebs, which advertises the SSID, channel and supported capabilities.
    - At the same time, it keeps listening to probe requests, which are like shouts in the air from the clients. 
    - If the AP catches a probe, it'll respond to it with the data that was being advertised up above. 
    - Once it the discovery phase (probe and response) is done, then the game of auth starts. The client shares an authentication request to the AP, which basically means that the client says hi, and the AP says you are good....for now. Post this, there is the association request, which basically is what let's the client into the BSS - Basic Service Set. The client gets an Association ID once that's done, and well, GG!
    - Oh and yeah, it runs the 4-way handshake logic with the clients and installs the keys into the drivers.     

---

## 2. DHCP and IP Address Assignment

See, that's cool and all. But the client connecting to the AP is like an amazon parcel in a warehouse - ONE AMONG MILLIONS. Somebody gotta identify the bastard man. And when it comes to the internet, there's only one way to do that - IP Address. 

But how do we assign the IP address to the connecting client? And more importantly, how do we make sure that the assigned IP Address is unique to the client? Well, people way more smart than you and me have come up with something called the Dynamic Host Configuration Protocol - a.k.a DHCP! For and AP, we use something called the DHCP Server. It basically assigns an IP to a connecting client, and when the connection closes, it takes back the IP to be potentially assigned to someone else. Now yes, there is a cache that holds the assigned IP for a while, just in case the client connects back, but we ain't talking about rn man. And yeah, technically, the AP doesn't run the DHCP server, it only acts as a gateway the the network which is hosted through a DHCP server, but for the sake of understanding, let's run with this for a moment, yes?

We can use a whole lot of DHCP utilities on the system but the one that I have worked with is called `dnsmasq`. It's quite simple to be honest, all I had to do was make a conf file and run a command. But y'all know me, I have to go deeper than just that! Here is how the dnsmasq runs - 
When a fresh client hops onto the AP, the DHCP dance begins. It’s called the **DORA process** (Discover → Offer → Request → Ack), and it goes something like this:
1. **Discover** → The client screams into the void: _“Hey, any DHCP servers out there? I need an IP!”_
2. **Offer** → Our friend `dnsmasq` replies: _“Sure bud, I can hook you up with 192.168.43.20, what do you say?”_
3. **Request** → The client says: _“Yes please, give me that exact address!”_    
4. **Ack** → The DHCP server signs off: _“Done. 192.168.43.20 is now officially yours (until your lease runs out).”_
All this back-and-forth happens over UDP (ports 67 and 68, if you’re a packet-sniffing nerd). And the beauty is, you can literally watch it in real-time with:

`tcpdump -i wlan0 port 67 or port 68 -n`

What makes `dnsmasq` neat is that it’s not just handing out IPs — it also doubles as a **DNS forwarder**. Meaning, when your client asks _“Where the hell is google.com?”_, dnsmasq doesn’t reinvent the wheel. It just passes the request upstream to a real DNS resolver and caches the result locally. Faster lookups, less load. Boom.

So at the end of this DHCP ritual, the client now has:

- A **private IP** (like 192.168.43.20)    
- A **default gateway** (your AP, usually 192.168.43.1)
- A **DNS server** (often the AP itself, via dnsmasq)
Congratulations — our parcel (the client) now has a clear shipping label slapped on. The warehouse (your AP) knows exactly where it belongs, and more importantly, how to get its packets back when replies start flowing in.   

---

## 3. NAT: The Real Magic

If the LTE modem gives your device an IP like `100.x.x.x` (CGNAT range), and your client has `192.168.43.20`, how do packets flow?
This is where **iptables NAT (MASQUERADE)** steps in:

`iptables -t nat -A POSTROUTING -o rmnet_data0 -j MASQUERADE`
- Translates the client’s private IP into the LTE interface IP. 
- Maintains a **connection tracking table** so replies come back to the right client.  
- Without NAT, packets would never reach the internet.  

Think of NAT as the “address translator” at the border between your LAN and the LTE world.

---

## 4. Putting It All Together

1. **hostapd** → configures AP, broadcasts SSID, handles WPA handshake.  
2. **dnsmasq / DHCP** → gives IPs to clients and sets default gateway.
3. **iptables NAT** → rewrites packets from clients to LTE interface IP.
4. **Kernel IP forwarding** (`/proc/sys/net/ipv4/ip_forward = 1`) → allows packets to actually flow between interfaces.

The end result? A seamless illusion of internet access.
From the client’s perspective:
`Laptop → 192.168.43.1 (hotspot gateway)        → LTE IP (100.x.x.x)        → Carrier NAT → Public Internet`

---

## 5. Why This Is Elegant
- Your hotspot doesn’t _really_ provide internet — it shares _its own_ internet.
- Wi-Fi is just the **access medium**, hostapd is the **door**, DHCP is the **ticket counter**, and NAT is the **passport office**.