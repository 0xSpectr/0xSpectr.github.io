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
im using this post more as a way for me to consolodate all of the knowledge i have learned from my networking studies, so if anything is wrong or misinterpiated please let me know


# DNS PROCESS
domain name resolution is a widley used protocol, its main goal is to convert human readable domain names like "facebook.com" into IP addresses
this way instead of trying to remeber "23.24.25.36" we can simply remember "facebook.com"
DNS runs over port 53/udp but can use TCP mainly for 
- Zone transfer(copying records between dns servers)   
- when the response is to large to fit into a single UDP packet
- DNS over TLS(DoT), DNS over HTTPS(DoT)

DNS is also used for much more then just storing servers IPs its also stores other records such as
- DKIM(stores the public key, used to verify the sender is who they say they are)   
- DMARC(tells the receiver what to do if there’s something wrong)   
- SPF(lists the valid servers that can send emails)   
- CNAMES(holds the name of another domain, used for forwarding)   
- MX(holds the IP of the mail servers) 
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
A content delivery network is a global network of distributed servers that cache commonly requested static(images, js, html etc) resources, this means that instead of connecting to a server in let’s say Japan we connect to the closest edge server near us that returns the the resource for us(if it doesn’t have the resource or it’s a non cachable resource the edge server reaches out to the origin server and pulls it) it does this through any cast routing, any cast routing is a routing method where multiple servers worldwide all broadcast the same IP address, then internet routing protocols such as BGP handle routing requests to the closest server(usually), because the closest server will have the least hops


# TCP Handshake
Now that we have the IP of the server(or edge server whatever) we can start communicating with it, but since most of the web uses TCP as its transport method we need to form a connection with it, this is what the tcp handshake handles its used to setup a reliable and steady connection with the server and make sure both sides are ready too send and receive data between each other, it has 3 main steps
1. First we send a TCP packet with the SYN flag set, this message also includes other data such as
   - MSS: the Maximum segment size defines the largest size in bytes the application data can be inside a single tcp segment, its calculated as MTU - tcp headers - IP headers, which on most machines will result in a MSS of 1460, MSS was built specifaclly to help stop fragmentation at the IP, the MSS is stored in the OPTIONS field of the tcp packet and is a 4 byte value
        - kind (1 byte): 2 → indicates Maximum Segment Size option
        - Length (1 byte): 4 → total length of the option in bytes (always 4)
        - MSS value (2 bytes): the actual maximum segment size in bytes
===============================================================
   - SACK: if the intiater of the connection supports SACK they will include a SACK permitted message, SACK is an extension to TCP and allows for more robust packet loss detection and handling, imagine during communication if the receiver received segments 1 2 4 5 6, its missing segment 3 so it keeps replying back to the sender with ACK 3 until the sender relises, the sender must now retransmit segment 3, 4 and 5, which is inefficant, SACK solves this by allowing the recevier to specify exactly what segment was lost and which ones they recevied, meaning the sender only has to resend the lost one, the SACK negotiation is also stored in the TCP options
   - Kind = 4 (SACK Permitted)
   - Length = 2
===============================================================
other options such as window scaling, timestamps, fast open exist but i wont go to in depth cause itll be to long
the sender will also setup their sequance numbers, in TCP sequance numbers are used to keep track of sent and recevied messages

2. the server receives this SYN packet and if its open to communicate(if ports closed we get a RST, if firewall blocks we get no response) the server will send back a SYN-ACK tcp message, this message also includes other data such as
   - SACK perm: if the server also supports SACK they will include it in their response, this way both sides now agree to use SACK for packet loss/handeling
   ===============================================================
   - MSS: the server will also include their own MSS, the smallest MSS value between them is chosen, but do note that this isnt static, either side of the connection can indivudally update their own MSS if something like PMTUD or some other form of it runs and detects a smaller value
   ===============================================================
and other options depending on what options the client sent, the server also setups their own sequance numbers

# TLS Handshake


