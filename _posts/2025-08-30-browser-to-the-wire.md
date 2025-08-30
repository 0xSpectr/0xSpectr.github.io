---
layout: post
title: "From the Browser To The Wire: What happens when you press enter"
date: 2025-08-30
categories:
  - networking
  - cybersec
---


# Intro
this post will be going over the full networking process of entering a url, all the way from DNS resolution to low level routing details
im using this post more as a way for me to consolidate all of the knowledge i have learned from my networking studies, so if anything is wrong or misinterpreted please let me know

overview
   - [DNS Process](#dns-process)
   - [CDN Overview](#content-delivery-network)
   - [TCP Handshake](#tcp-handshake)
   - [TLS Handshake](#tls-handshake)
   - [HTTP & Encapsulation](#http-&-encapsulation)
   - [Routing](#routing)


# DNS Process
domain name resolution is a widely used protocol, its main goal is to convert human readable domain names like "facebook.com" into IP addresses
this way instead of trying to remember "23.24.25.36" we can simply remember "facebook.com"
DNS runs over port 53/udp but can use TCP mainly for 
- Zone transfer(copying records between dns servers)   
- when the response is to large to fit into a single UDP packet
- DNS over TLS(DoT), DNS over HTTPS(DoH)


DNS is also used for much more then just storing servers IPs its also stores other records such as
- AAAA(stores IPv6 addresses)
- DKIM(stores the public key, used to verify the sender is who they say they are)   
- DMARC(tells the receiver what to do if there’s something wrong)   
- SPF(lists the valid servers that can send emails)   
- CNAMES(holds the name of another domain, used for forwarding)   
- MX(holds the host names of mailservers) 
- PTR(used for reverse DNS lookups, stores the domain name instead of IP)

First we enter Facebook.com into a url bar
our browser will now start the DNS process, which has 5 main steps

1. First the browser checks the systems local host files, this is a file on disk(/etc/hosts on linux, C:\Windows\System32\Drivers\etc\hosts on windows), this file stores hardcoded domain to IP mappings, it takes precedence over all else, its commonly used inside LANs to allow for static entrys of server   
2. Browsers Local cache, all browsers keep a cache of recetnly converted domain names, this way we dont need to intiate requests everytime we want to visit a website we commonly use   
3. Systems Local Cache, our system also keeps a cache of recently converted domain names, it works the same as above   
4. If the IP wasnt found in any of the above we now intiate the outbound DNS process, our system constructs a DNS packet and sends it the primary DNS server in our systems network configuration(Learned via DHCP or manually entered), ill use 8.8.8.8
5. the DNS server receives this DNS request, this server is known as a recursive resolver, meaning it takes over and handles the rest of the resolution for us
    - First the resolver reaches out to a root dns server, there are 13 of these servers(a.root-servers.net-m.root-servers.net) but they are split across thousands of nodes worldwide and use anycast routing(see CDN section for anycast explanation), root servers handle refering us to the correct TLD server that handle our domain(.com, .cloud etc)
    - the resolver then recevies the IP of the correct TLD server from our domain and then sends the DNS request to it, TLD servers handle looking up our domain and returning us a list of authoratative nameservers
    - finally the resolver receives the list of Authoritative nameservers, it then chooses one and sends the final DNS request to it, the Authoritative nameserver then handles returning us the value we need in most cases its the A record which stores the IPv4 address

Do Note that all of the above servers keep their own cache of recently converted domains and at each step it checks the cache first, most requests for big sites such as google.com, facebook.com etc will stop at the resolver, just didnt mention the cache at each server for brevity

# Content Delivery Network
Facebook and almost all large sites these days use a CDN so ill quickly explain it
A content delivery network is a global network of distributed servers that cache commonly requested static(images, js, html etc) resources, this means that instead of connecting to a server in let’s say Japan we connect to the closest edge server near us that returns the the resource for us(if it doesn’t have the resource or it’s a non cachable resource the edge server reaches out to the origin server and pulls it) it does this through any cast routing, any cast routing is a routing method where multiple servers worldwide all broadcast the same IP address, then internet routing protocols such as BGP handle routing requests to the closest server(usually), because the closest server will have the least hops, this also has the added bonus that users never directly interact with the origin server, giving us a little bit of security 


# TCP Handshake
Now that we have the IP of the server(or edge server whatever) we can start communicating with it, but since most of the web uses TCP as its transport method we need to form a connection with it, this is what the tcp handshake handles its used to setup a reliable and steady connection with the server and make sure both sides are ready too send and receive data between each other, it has 3 main steps
1. First we send a TCP packet with the SYN flag set, this message also includes other data such as
   - MSS: the Maximum segment size defines the largest size in bytes the application data can be inside a single tcp segment, its calculated as MTU - tcp headers - IP headers, which on most machines will result in a MSS of 1460, MSS was built specifaclly to help stop fragmentation at the IP, the MSS is stored in the OPTIONS field of the tcp packet and is a 4 byte value
        - kind (1 byte): 2 → indicates Maximum Segment Size option
        - Length (1 byte): 4 → total length of the option in bytes (always 4)
        - MSS value (2 bytes): the actual maximum segment size in bytes

   - SACK: if the initiator of the connection supports SACK they will include a SACK permitted message, SACK is an extension to TCP and allows for more robust packet loss detection and handling, imagine during communication if the receiver received segments 1 2 4 5 6, its missing segment 3 so it keeps replying back to the sender with ACK 3 until the sender relises, the sender must now retransmit segment 3, 4 and 5, which is inefficient, SACK solves this by allowing the receiver to specify exactly what segment was lost and which ones they recevied, meaning the sender only has to resend the lost one, the SACK negotiation is also stored in the TCP options
        - Kind = 4 (SACK Permitted)
        - Length = 2
   - Timestamp: another very common tcp option that is seen in the TCP handshake are timestamps, timestamps serve multiple purposes such as
        - RTT calculation, timestamps allow for milliseconds accurate RTT calculation as all packets include the timestamp of when it was sent, the receiver then can just do TSecr_echoed - current_time = RTT
        - detecting duplicate/stale packets
     the way it works is that if the sender has timestamps enabled they will include 2 values in the tcp option, TSval which is the current timestamp and TSecr which is 0 for now since its used to echo back the senders TSval 

2. the server receives this SYN packet and if its open to communicate the server will send back a SYN-ACK tcp message, this message also includes other data such as
   - SACK perm: if the server also supports SACK they will include it in their response, this way both sides now agree to use SACK for packet loss/handling
   - MSS: the server will also include their own MSS, the smallest MSS value between them is chosen, but do note that this isnt static, either side of the connection can individually update their own MSS if something like PMTUD or some other form of it runs and detects a smaller value
   - timestamp: if the server has timestamps enabled the SYN-ACK packet will also include a timestamp, TSval will be the timestamp of when the server replys and TSecr will echo the TSval the client sent in the SYN packet

3. finally the client will send back a ACK packet, this final packet finalises the tcp handshake and the connection is now established 

Do note that there are more tcp options such as fast open, window scaling etc i just didn't add them for brevity
also not all these options are present in every handshake, but they are still very common and enabled on most machines
during the process each side sets up their intial sequence numbers, sequence numbers are used by tcp for segment re ordering, packet loss detection, keeping track of data etc

# TLS Handshake
Once the TCP connection is established, we need to set up a secure encrypted connection using HTTPS. This is done via the TLS handshake:

1. Client Hello: The client sends a message including:
   - Supported cipher suites
   - Supported TLS versions
   - Client random (32-byte value used in key derivation)
   - Other handshake parameters

2. Server Hello: The server responds with:
   - Server’s certificate (for authentication)
   - Chosen cipher suite
   - Chosen TLS version
   - Server random

3. Certificate validation & key generation:
   - The client validates the server’s certificate. 
   - Both client and server then generate ephemeral keys for key exchange (TLS 1.3 uses ECDHE; RSA is only used for certificate signatures).

4. Finished messages:
   - The server sends a **Finished** message containing a hash of all previous handshake messages, encrypted with the session key.
   - The client decrypts and verifies it, then responds with its own **Finished** message.
   - The server decrypts and verifies the client’s Finished message.

At this point, the handshake is complete and the encrypted session is established.
Note: 
   In TLS 1.3, the key exchange is included in the ServerHello, and the exact key generation method depends on the chosen cipher suite and TLS version. Modern browsers almost always use ephemeral ECDH for key exchange.

# HTTP & Encapsulation
NOTE:
   HTTP/3 uses QUIC, a modern transport protocol built on top of UDP instead of TCP like previous HTTP versions, QUIC provides the reliability and congestion control of TCP but without its overhead. It also includes built-in TLS, supports 0-RTT handshakes, enables better multiplexing through individual streams, and solves the head-of-line blocking issue that TCP can face.
   For simplicity, this blog focuses on HTTP/1.1–2, which uses TCP as its transport method.
   If you would like to read more about [QUIC](https://www.auvik.com/franklyit/blog/what-is-quic-protocol/)

Now that the secure TLS connection is established, our browser can finally request resources from the server using HTTP (Hypertext Transfer Protocol). HTTP is a Layer 7 application protocol that powers the web. It allows us to request files, submit forms, manage sessions, and interact with web applications.
HTTP uses methods to define the operaiton we would like to do to such as:
   - GET: request a resource
   - POST: upload a resource(such as files)
   - PUT: update a resource(such as usrr detsils)
   - DELETE: delete a resource
for us we would like to request a resource so we senda GET request, http also uses headers which allow us to send other details to the server some common ones are
   - User-Agent: a string thay identifies our device type, such as a mobile phone, desktop etc, it also includes whag browser we are using, the version and more
   - Accept: defines the type of data we will accept, such as html, css etc
   - Cookie: a small string that allows the server to identify us across requests, because HTTP is stateless cookies allow the server to remember us, they are automatically included in requests by the browser(depening on the cookies settings and requests origin) and is what allows us to login once then return later and stay logged in
   - Host: specfies the domain name of the server, required in 1.1 requests
so for examples our request to facebook.com will look like
   GET / 1.1
   Host: facebook.com
   User-Agent: <whatever>
   Accept: text/html, image/jpeg
   cookie: <whatever>

Now our browser constructs the GET requests and the encapsulation process happens now
i will be using the OSI model for this explanation so i can explain each layer more in depth, but the modern intetnet uses the TCP/IP model which merges layers 7,6 and 5 into a single layer

Now the encapsulation process takes place, the above was layer 7 application

1. Next it gets passed down to layer 6 presentation, this layer handles the presentation of the data, stuff such as encoding, compression and encryption etc, here any dangerous characters in the url get to encoded according the url encoding spec, and then the GET requests is encrypted with session  using the key that was created during the TLS handshake
<skipping session layer since its just not important in the modern web>.   

2. Next its passed down the transport layer, this layer handles the end to end communication of data, re ordering, segmentation, packet loss detection etc, here the data is encapsulated into a segment and a TCP header is added this includes;
    - Source port
    - Destination port
    - checksum(used for corrupt/tampered segment detection)
    - TCP flags if any(psh, urg etc)
    - sequence number
    - ACK number(next byte we exepct from server)
    - Window size(how much free space is in our buffer, used by the server for flow control)

3. Next its passed down to the IP layer, this layer handles the routing of data, fragmentation, reassembly etc, at this layer the tcp segment is again encapsulated into a IP packet and an IP header is added onto it which includes
    - Source IP(our private IP for now)
    - destination IP(servers public IPv4)
    - checksum(same as above)
    - need fragmentation flag(only needed if total size of packet is larger then systems MTU, usually avoided at all costs)
    - version(IP version)
    - protocol(protocol in use, TCP for us)
    - TTL(used to stop routing loops, gets decremented at every hop, if hits 0 thr packet is dropped)
    - and more

4. Next its passed down to the Data link layer, this layer handles the Hop-by-hop of data, basic corruption checks etc, here the IP packet is encapsulated into a ethernet frame and a etherner header is added whjch includes:
    - source MAC
    - Destination MAC
    - FCS(used for error detection,its a checksum that's calculated ober the etherner header + payload)
    - EtherType(specifys the encapsulated protocol, IPv4, IPv6, ARP etc)
    - VLAN ID(not always)

NOTE:
   if this was HTTP/3 using QUIC the transport layer would instead apply a UDP header, which is much simplier then a TCP header

#Routing





