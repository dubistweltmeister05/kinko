[[DHCP]]

Well, Dynamic Host Control Protocol is a network protocol that is used to assign IP addresses automatically to any device that is connecting to the configured network. This allows any and all devices to connect to the network and receive all necessary information, such as the IP address, the DNS server address, the default gateway, and the subnet mask. 

The main components of DHCP are as follows - 
1. DHCP Server - This is the part of the config that holds the pool of IP addresses and is responsible for maintaining the whole configuration info that is to be assigned to the connecting devices. 
2. DHCP Client - The device that receives the config into and communicated with the server. This is any device that connects to the network and needs to use the internet - per say.
3. DHCP Relay - This is what enables the talking between the server and the client.
4. IP Address pool - This is the set of addresses that are given to the server, from which the incoming clients are assigned an address. 
5. Subnets - These are small, static sections of the IP address pool that are used to control the allocation of the IP addresses. You use these to make sure that a set of devices can remain localized within a few IP addresses. 
6. Lease - The time for which the info sent by the server is valid. Once it expires, the client needs to request another lease from the server. This is configurable for every network.
7. DNS Server - The DHCP sever can also provide a DNS Server to their client, allowing them to resolve domain names to their IP Addresses. This also allows for blocking of a few IP addresses at the network level, for example - any AD requests to known advertisement servers. 
8. Default Gateway - This is the device that receives packets from the clients when the destination of the transmission is outside the network. 
9. Options - Additional configuration data that may or may not be provided.
10. Failover - the DHCP server can be configured for the event of a failure, where 2 servers work at the same time to provide redundancy. 

There is a particular packet format the the DHCP data should follow, with each field playing a crucial part in the whole exchange. 

![[Pasted image 20250905154019.png]]

1. Operation Code - 1 for messages sent by the server, 2 for messages sent by the client
2. Hardware Type - Used to specify the type of LAN being used by the system.
3. Hardware Length Address -MAC address, used to define the length of the hardware address in the Charaddr field. 6 if ethernet is used.
4. Hops - The number of relay agents that have been configured. 
5. Transaction Identifier - Used by clients to match server responses with the previous requests
6. Seconds -Time elapsed since the start of the process. 
7. Flags - Called as the broadcast bit, this is set to 1 if the message to the client must be broadcasted.
8. Ciaddr - Client's IP address, set by the client to confirm a valid IP.
9. Yiaddr - Client's IP address, set by the server to assign the client an IP
10. Siaddr - IP address of the next server that the client needs to connect to. 
11. Giaddr - Gateway IP, this is filled by the relay agent, with the address of the interface through which the DHCP message has been received. 
12. Chaddr - Client's hardware address (MAC address or Level 2 address)
13. Sname - Name of the next server for the client to use in the config process.  
14. File - Name of the file for the client to request from the next serve

# Working of DHCP

The whole process happens in the form of messages. The flow is as follows
## 1. DHCP Discover message 
- This is the first message that is sent by the client out on the network, with a target address of 255.255.255.255 and a source address of 0.0.0.0. 
- This is done to see if there are any servers available on the network that are valid.
- It might contain some more information, like a subnet mask, DNS, domain name, etc.
- It is broadcasted to all the devices on the network.
## 2.DHCP Offer message - 

- The DHCP server shall reply/respond to the client that sent the discover message. This holds the released IP address and other config parameters that have been set.
- It is broadcasted by the server.
- If there are multiple servers, the client will accept the first message that it receives. Also, there is a server ID that is mentioned in the packet that is used to identify the ID of the server. 
## 3. DHCP Request message - 
- The Client receives the DHCP offer message from the DHCP server that replied/responded to the DHCP discover message.
- Then the client compares the request that has been offered and chooses the server that it wants to use. The client sends the request message, indicating the chosen server. 
- This is again, broadcasted to the whole network, to let everyone know which server has been chosen.
## 4. DHCP Acknowledgement message -
- This is sent by the server once it receives the request message. The server marks the IP address as leased, and those who haven't been selected simply add the IP back to their address pool. 
- Now, a DHCP ACK is sent by the server, which may hold additional config options.
- The client may use the IP address and configuration parameters. It will use these settings till its lease expires or till the client sends a DHCP Release message to the server to end the lease.

Once this is done, well, bob's your uncle!