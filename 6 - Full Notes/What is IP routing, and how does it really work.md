[[Networking]]

This is nothing but a way of receiving IP packets on an interface and routing them to another interface, but all done via the kernel. 

An interface shall receive some packets, look up a routing table, and send them to the route that was defined already in the said table. Some key-capabilities of this are - 
- Inter-Network Routing - This helps forward traffic between distinct subnets and networks. 
- Network Address Translation - This is used to forward packets from private networks to public ones. 
- VPN - Receive VPN traffic and forward it to the subnets.
- Container Networking - Enable containers to communicate between hosts by routing through them.

## So, how the hell does IP forwarding work in the kernel? 

Well, there are 2 important parts of doing this - 
1. Kernel IP layer. 
   This is what actually handles the forwarding of the packets from an interface to another one. It maintains the routing tables and forwards the packets that it receives according to the routes that have been defined. 
   
   Some aspects of this are - 
	- Static routes are supported via routing tables, and dynamic routing is supported via routing protocols, BGP, OSPF, etc. 
	- Routing decisions are made based on longest prefix match of the destination IP against the routing table.
	- Common Linux dynamic routing protocols include OSPF, BGP, RIP, IS-IS. Static routes can also be used. 
	- Forwarding is handled in a kernel process completely separate from user space networking.
2. Netfiltering 
	While the kernel forwarding path routes packets between interfaces, Netfilter provides various points to hook into the packet flow for filtering, mangling, NAT, and more.
	
	Some capabilities enabled by Netfilter:

	- **iptables** – Filter packets and forward as per defined firewall rules.
	- **NAT** – Source NAT outgoing packets or destination NAT inbound ones.
	- **Packet mangling** – Modify headers like TOS bits or ports.
	- **Raw table access** – Process packets in user space.
 