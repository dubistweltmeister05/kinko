[[IPT - 4 Network Address Translation Introduction]]

## What is a NAT and how is it used

A Network Address Table is used to allow a host/several hosts to connect to the network in a single IP address. For example, let's say we have a local network consisting of 5-10 clients. We set their default gateways to point through the NAT server. Normally the packet would simply be forwarded by the gateway machine, but in the case of a NAT server it is a little bit different.

NAT servers translate the source and destination addresses to a different address. It receives the packet, and re-writes the Source and/or destination address and then re-calculates the checksum of the packet. 

One of the most common usage of NAT is Source NAT, called as SNAT. Here, we don't want to have a real, public IP for the clients that we have connected. So, we use one of the private IP ranges for our local network (for example, 192.168.1.0/24), and then we turn on SNAT for our local network. SNAT will then turn all 192.168.1.0 addresses into it's own public IP (for example, 217.115.95.34). This way, there will be 5-10 clients or many many more using the same shared IP address.

There is something called as DNAT as well, which is really helpful while setting up servers. We can get an IP packet from the public network, and DNAT shall re-write it to a private network's IP address, helping us establish an IMPENETRABLE Firewall between 2 local networks. The reply traffic is automatically translated back to the original IP address, obviously. 

Now, there's 2 types of NAT that can be run in Linux. They are Fast-NAT and Netfilter-NAT. Let's take a look - 

#### Fast NAT
It allows specific LAN hosts to skip the inspecting by CPU and go to the fast NAT path directly. Which means, the router will forward the traffic from a particular LAN to the chosen WAN directly. This function will reduce the CPU loading and speed up the performance of for the NAT sessions.
#### Netfilter NAT - TODO

## Caveats using NAT
Well, some applications may not work, like, at all.

The second and smaller problem is applications and protocols which will only work partially. These protocols are more common than the ones that will not work at all, which is quite unfortunate, but there isn't very much we can do about it as it seems. If complex protocols continue to be built, this is a problem we will have to continue living with. Especially if the protocols aren't standardized.

The third, and largest problem, in my point of view, is the fact that the user who sits behind a NAT server will not be able to run their own server. This is because when you initiate a connection out to the internet, NAT dynamically creates a translation mapping (private → public). But if an external client tries to initiate a connection back in, there is no such mapping, and the NAT doesn’t know which internal host to forward the packet to.

This can be solved by configuring **static mappings** (also called port forwarding or DNAT). In this case, the NAT is told in advance:

Public_IP:Port → Private_IP:Port

so that incoming connections can always be directed to the correct internal server. With this static mapping, running a server behind NAT becomes technically possible.