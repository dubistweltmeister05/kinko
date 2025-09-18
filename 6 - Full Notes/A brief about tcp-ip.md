[[IPT - 2 TCP-IP repetition]]

## The layers of TCP/IP
Come on son. We know this!

#### OSI Model
1. Application layer
    
2. Presentation layer
    
3. Session layer
    
4. Transport layer
    
5. Network layer
    
6. Data Link layer
    
7. Physical layer

#### TCP/IP Model
1. Application layer
    
2. Transport layer
    
3. Internet layer
    
4. Network Access layer

## IP Characteristics
The IP protocol is responsible for telling where to go, what to communicate and all that important info to the packets that we send, over a router, a modem, a switch, etc. Safe to say, it is the backbone of this world.

The IP protocol encapsulates the Transport layer packet with information about which Transport layer protocol it came from, what host it is going to, and where it came from, and a little bit of other useful information. All of this is, of course, extremely precisely standardized, down to every single bit.

There are a few basics that the protocol should be able to handle on it's own. Like, it should be able to define the datagram which is the next building block created by the transport layer. The IP protocol has also defined the addressing system that we use to talk over the interwebs. This is the same IP address that we keep harping on about. 

Further, the IP protocol should be able to de/encapsulate the IP Datagram, and send or receive the datagram from either the Network access layer, or the transport layer. It is also responsible for routing any packets that it receives or sends out. Lastly, it must fragment and reassemble any datagram that has previously been fragmented, or that needs to be fragmented to fit in to the packet size of this specific network hardware topology that we are connected to.

Oh, and yes - the IP is a connectionless protocol. This means the the IP does not care or negotiate about any physical connections. That, is handled by anything that is on top of the IP protocol. Say hello to TCP/UDP. Also - why connectionless? Overhead. That's it. 

This makes it hella unreliable too btw. IP does not give a single flying fuck about the reception of the packet. It only cares about sending it out to what we have configured it for. 

## IP Headers

There is a fuck ton of info in the header, and it's for good reasons. There is a structured breakup of the bit fields of the IP header, which looks as follows - 
![[Pasted image 20250915121036.png]]

Let's break it down -
1. Version [0-3]- IPv4 or IPv6 basically.
2. IHL [4-7]- How long the header is, in units of 32-bit words.
3. Type Of Service [8-15] 
4. Length [16-31]- How long the packets are, in units of octets(bytes). Max size is 65535. 
5. ID [32-46]- Used to re-fragment something that was messed up.
6. Flags [47-49]- Flags about the partition. The first bit is reserved, don't touch that. The second is set to 0 if the packet might be fragmented, and to 1 if the packet is not. The third and last bit can be set to 0 if this was the last fragment, and 1 if there are more fragments of this same packet. 
7. Offsets [50-63]- Shows where in the datagram that this packet belongs. The fragments are calculated in 64 bits, and the first fragment has offset zero.
8. Time to Live [64-72]- The TTL field tells us how long the packet may live, or rather how many "hops" it may take over the Internet. Every process that touches the packet must remove one point from the TTL field, and if the TTL reaches zero, the whole packet must be destroyed and discarded. This is basically used as a safety trigger so that a packet may not end up in an uncontrollable loop between one or several hosts. Upon destruction the host should return an ICMP Time exceeded message to the sender.
9. Protocol [73-80]- Bits to indicate the protocol that is to be used next.
10. Checksum [81-96]- CRC that is re-calculated whenever the host changes.
11. Source address [97-128]- This is the source address field. It is generally written in 4 octets, translated from binary to decimal numbers with dots in between.
12. Destination address [129-160]- The destination address field contains the destination address, and what a surprise, it is formatted the same way as the source address.
13. Options [161 - 192/478]-  The options field is not optional, as it may sound. Actually, this is one of the more complex fields in the IP header. The options field contains different optional settings within the header, such as Internet timestamps, SACK or record route route options. Since these options are all optional, the Options field can have different lengths, and hence the whole IP header. However, since we always calculate the IP header in 32 bit words, we must always end the header on an even number, that is the multiple of 32. The field may contain zero or more options.
14. Paddings []- 0s used to align the packet to a 32-bit boundary.

## TCP Characteristics
This lies on top of IP. It is a stateful protocol and has built-in functions to see that the data was received properly by the other end host. The main goals of TCP are - 
- To see that data is reliably received and sent.
- The data is transported between the Internet layer and Application layer correctly.
- The packet data reaches the proper program in the application layer.
- The data reaches the program in the right order.
This, is possible only because of the TCP headers of the packets.

Data is looked at by TCP as a continuous stream with a start and a stop signal. The `SYN (3-way handshake)` signal indicates that there is a new connection to be opened, with one packet being sent with the SYN bit set. The other end responds with a `SYN/ACK` or `SYN/RST` signal to let the initiator know about the connection status, `ACK` meaning acceptance and `RST` being denial. If the client receives a SYN/ACK packet, it once again replies, this time with an ACK packet. At this point, the whole connection is established and data can be sent. During this initial handshake, all of the specific options that will be used throughout the rest of the TCP connection is also negotiated, such as ECN, SACK, etc.

While the data stream is alive, we have further mechanisms to see that the packets are actually received properly by the other end. This is the reliability part of TCP. This is done in a simple way, using a Sequence number in the packet. Every time we send a packet, we give a new value to the Sequence number, and when the other end receives the packet, it sends an ACK packet back to the data sender. The ACK packet acknowledges that the packet was received properly. The sequence number also sees to it that the packet is inserted into the data stream in a good order.

Once the connection is closed, this is done by sending a FIN packet from either end-point. The other end then responds by sending a FIN/ACK packet. The FIN sending end can then no longer send any data, but the other end-point can still finish sending data. Once the second end-point wishes to close the connection totally, it sends a FIN packet back to the originally closing end-point, and the other end-point replies with a FIN/ACK packet. Once this whole procedure is done, the connection is torn down properly.

As you will also later see, the TCP headers contain a checksum as well. The checksum consists of a simple hash of the packet. With this hash, we can with rather high accuracy see if a packet has been corrupted in any way during transit between the hosts.

## TCP Headers
![[Pasted image 20250915142445.png]]

1. Source Port []- 
2. Destination Port []- 
3. Sequence Number []- 
4. Acknowledgement Number []- 
5. Data Offset []- 
6. Reserved []- 
7. CWR []- 
8. ECE []- 
9. URG []- 
10. ACK []- 
11. PSH []- 
12. RST []- 
13. SYN []- 
14. FIN []- 
15. Window []- 
16. Checksum []- 
17. Urgent Pointer []- 
18. Options []- 
19. Padding []- 
20. Data []- 


