[[Networking]]

# Well, what the hell is a DNS server? 
A Domain Name Service is like the phonebook of the internet. We don't access the web via IP addresses, do we? That'll be inconvenient a, like, imagine typing out `8.8.8.8` every time that you need to open Google to look something up, lmao. Typing `google.com` is quite convenient, isn't it. That service is provided by the DNS server, which does nothing but match the text that we type to an IP address. 

Seems simple enough, but how does it work? Like, how does the matching of the text with the IP address happen in the first place?

Let's dig through deeper. 

The whole operation is nothing but converting the domain name to an IP address. That's it. An IP address is given to each device on the Internet, and that address is necessary to find the appropriate Internet device - like a street address is used to find a particular home. When a user wants to load a webpage, a translation must occur between what a user types into their web browser (`example.com`) and the machine-friendly address necessary to locate the `example.com` webpage.

![[Pasted image 20250910102104.png]]
Essentially, there are 4 DNS servers that the computer must interact with in order to load the webpage - 
1. ***DNS recursor resolver***
   This is the first component that a request that is made for DNS resolution shall pass through. Think of it as a middleman between a client and the actual DNS nameserver. When a request is received, the resolver shall either respond with the cached data that it has, or will forward the request to the nameserver, followed by another request to a TLD nameserver, and then one last request to an authoritative nameserver. After getting something from the authoritative nameserver, it will cache the response, should there be a repeat request for the same resource in the future.  
2. ***Root Nameserver***
   There are 13 root nameservers in this world. This does not mean that there are 13 devices acting as RNS, and yes, that is what I first thought as well. They are 13 labels, A-root through M-Root, with each root pointing to a few hundred physical devices. Why 13? This is based on the days of UDP, which had a 512-byte size limit on the packet. When a DNS query is raised, it returns a list of the available root nameservers that are available, and, well, only 13 could fit within the 512-byte packet! 
   
   These 13 nameservers are known to all recursor resolvers. They are the first ones that accept the resolver's DNS query which includes the domain name, and responds by directing the query to a TLD server, based on the extension of the domain (.org, .in, .com, .net).  They are overseen by a nonprofit called the Internet Corporation for Assigned Names and Numbers (ICANN).
3. ***TLD Nameserver***
   The Top Level Domain nameserver is used for managing and providing information about domain names that share the same **Top-Level Domain (TLD)**, such as _.com_, _.org_, _.uk_, or _.edu_. The use of this is to group domains with a similar extension, so that it can be passed on to the correct authoritative nameserver. There are 2 types of TLD nameservers - the generic ones, like .com, .org, .net, .edu, and .gov; and the country specific ones, like  .uk, .us, .ru, and .jp. There is actually a third category for infrastructure domains, but it is almost never used. This category was created for the .arpa domain, which was a transitional domain used in the creation of modern DNS; its significance today is mostly historical.
4. ***Authoritative Nameserver***
   When a recursive resolver receives a response from a TLD nameserver, that response will direct the resolver to an authoritative nameserver. The authoritative nameserver is usually the resolver’s last step in the journey for an IP address. The authoritative nameserver contains information specific to the domain name it serves (e.g. google.com) and it can provide a recursive resolver with the IP address of that server found in the DNS A record, or if the domain has a CNAME record (alias) it will provide the recursive resolver with an alias domain, at which point the recursive resolver will have to perform a whole new DNS lookup to procure a record from an authoritative nameserver (often an A record containing an IP address). 

# So, what are the steps involved in a typical DNS lookup? 
Typically, a DNS lookup (or rather, the response to a particular lookup) is cached, so the following discussion is under the assumption that the request is a FRESH one, so no cheating-I mean caching is involved. 

There are 8 usual steps involved in resolving a lookup request - 
1. The user types a domain name "xyz.com" hits enter - which makes it's way through to the DNS recursor. 
2. The resolver then queries the root nameserver (DBMS jindabaad bhai) by the domain request.
3. The RNS responds with a direction to a TLD nameserver, which is essentially the server that handles a particular domain (.com for this case).
4. The resolver then makes a request to the .com TLD.
5. The TLD server then responds with the IP address of the domain’s nameserver, xyz.com .
6. Lastly, the recursive resolver sends a query to the domain’s nameserver.
7. The IP address for example.com is then returned to the resolver from the nameserver.
8. The DNS resolver then responds to the web browser with the IP address of the domain requested initially.

Once the 8 steps of the DNS lookup have returned the IP address for example.com, the browser is able to make the request for the web page:

9. The browser makes a HTTP request to the IP address.
10. The server at that IP returns the webpage to be rendered in the browser (step 10).

![[Pasted image 20250910112144.png]]

## What are the types of DNS queries?

In a typical DNS lookup three types of queries occur. By using a combination of these queries, an optimized process for DNS resolution can result in a reduction of distance traveled. In an ideal situation cached record data will be available, allowing a DNS name server to return a non-recursive query.

#### 3 types of DNS queries:

1. **Recursive query** - In a recursive query, a DNS client requires that a DNS server (typically a DNS recursive resolver) will respond to the client with either the requested resource record or an error message if the resolver can't find the record.
2. **Iterative query** - in this situation the DNS client will allow a DNS server to return the best answer it can. If the queried DNS server does not have a match for the query name, it will return a referral to a DNS server authoritative for a lower level of the domain namespace. The DNS client will then make a query to the referral address. This process continues with additional DNS servers down the query chain until either an error or timeout occurs.
3. **Non-recursive query** - typically this will occur when a DNS resolver client queries a DNS server for a record that it has access to either because it's authoritative for the record or the record exists inside of its cache. Typically, a DNS server will cache DNS records to prevent additional bandwidth consumption and load on upstream servers.

## What is DNS caching? Where does DNS caching occur?

The purpose of caching is to temporarily stored data in a location that results in improvements in performance and reliability for data requests. DNS caching involves storing data closer to the requesting client so that the DNS query can be resolved earlier and additional queries further down the DNS lookup chain can be avoided, thereby improving load times and reducing bandwidth/CPU consumption. DNS data can be cached in a variety of locations, each of which will store DNS records for a set amount of time determined by a time-to-live (TTL)

#### Browser DNS caching

Modern web browsers are designed by default to cache DNS records for a set amount of time. The purpose here is obvious; the closer the DNS caching occurs to the web browser, the fewer processing steps must be taken in order to check the cache and make the correct requests to an IP address. When a request is made for a DNS record, the browser cache is the first location checked for the requested record.

In Chrome, you can see the status of your DNS cache by going to chrome://net-internals/#dns.

#### Operating system (OS) level DNS caching

The operating system level DNS resolver is the second and last local stop before a DNS query leaves your machine. The process inside your operating system that is designed to handle this query is commonly called a “stub resolver” or DNS client. When a stub resolver gets a request from an application, it first checks its own cache to see if it has the record. If it does not, it then sends a DNS query (with a recursive flag set), outside the local network to a DNS recursive resolver inside the Internet service provider (ISP).

When the recursive resolver inside the ISP receives a DNS query, like all previous steps, it will also check to see if the requested host-to-IP-address translation is already stored inside its local persistence layer.

The recursive resolver also has additional functionality depending on the types of records it has in its cache:

1. If the resolver does not have the A records, but does have the NS records for the authoritative nameservers, it will query those name servers directly, bypassing several steps in the DNS query. This shortcut prevents lookups from the root and .com nameservers (in our search for example.com) and helps the resolution of the DNS query occur more quickly.
2. If the resolver does not have the NS records, it will send a query to the TLD servers (.com in our case), skipping the root server.
3. In the unlikely event that the resolver does not have records pointing to the TLD servers, it will then query the root servers. This event typically occurs after a DNS cache has been purged.