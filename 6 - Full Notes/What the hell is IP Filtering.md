[[IPT - 3  IP filtering introduction]]

## What the fuck is an IP filter?
An **IP filter** is a rule (or set of rules) that decides what to do with network packets based on their **IP header** and sometimes higher-level info (protocol, ports). It Lies mainly in the layer 2 of the TCP/IP stack, although the iptables utility can work in the 3rd layer as well. Iptables is not able to follow a trail of data, or puzzle some together, it'll be too processor and memory intensive for it to do that. 

But, it does keep track of packets and see if they are for the same stream (via sequence numbers, port numbers, etc.) almost exactly the same way as the real TCP/IP stack. This is called connection tracking, and thanks to this we can do things such as `Destination and Source Network Address Translation` (generally called DNAT and SNAT), as well as state matching of packets.

## How to plan an IP Filter
The placement of the firewall is a good place to start at. One of the better ideas would be to plan one at the gateway of the local network and the internet. 

One way to go about this is to make a De-Militarized Zone (DMZ). A DMZ is a small physical network with servers, which is closed down to the extreme. This lessens the risk of anyone actually getting into the machines in the DMZ, and it lessens the risk of anyone actually getting in and downloading any trojans etc. from the outside. 

The reason that they are called de-militarized zones is that they must be reachable from both the inside and the outside, and hence they are a kind of grey zone (DMZ simply put).

Now, to set-up the firewall, there are a couple options. The most common strategies are "Accept everything apart from what we tell you to drop", and "Reject everything apart form what we tell you to accept". Pretty self-explanatory, isn't it?

Mostly, we are into the drop policy, since that list is usually shorter to maintain. This means that the firewall is more secure by default, but it may also mean that you will have much more work in front of you to simply get the firewall to operate properly.

It may also be a good idea to apply layered security measures, which we have actually already discussed partially so far. What is meant with this, is that you should use as many security measures as possible at the same time, and don't rely on any one single security concept. Having this as a basic concept for your security will increase security tenfold at least.