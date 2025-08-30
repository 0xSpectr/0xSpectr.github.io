---
layout: post
title: "From the Browser To The Wire: What happens when you press enter"
date: 2025-08-30
categories:
  - networking
  - cybersec
---


# Intro
this post will be going over the full process of entering a url, all the way from DNS resolution to low level routing details
this post is being used more as a way for me to consolodate all of the knowledge i have learned from my networking studies, so if anything is wrong or misinterpiated please let me know


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
