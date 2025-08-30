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
4. 

The dns server receives this, this server is also known as a recursive resolver meaning it takes over and handles the rest for us
First the resolver reaches out to a root dns server, there are 13 root dns servers(distributed over 1000s of servers world wide), these servers handle looking the TLD  inside the FQDN the resolver sent and returning the IP of the TLD server that handles the .com TLD, the root server then sends back the IP to the resolver
Now the resolver sends another dns message to the returned IP, this server is known as a TLD server, its holds all the information about the passed TLD, its stores the name servers for whatever registerar the domain was bought from, these name servers are then returned back to the resolver
Finally the resolver chooses one of the nameserver, these servers also known as authoritative dns servers actually store all the dns records like MX, A, AAAA etc, finally the authoritative returns the final IP of the domain we requests
NOTE
All of the mentioned servers also have a cache, they each individually check their cache and when a request is sent to them, I just didn’t specify it for each server for brevity, for big domains like Google their IPs are usually cached and found at step 1 at the resolver
