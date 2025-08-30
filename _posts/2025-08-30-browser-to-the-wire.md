---
layout: post
tite: "From the Browser To The Wire: What happens when you press enter
date: 2025-08-30
catergories: networking, cybersec
---


# Intro
this post will be going over the full process of entering a url, all the way from DNS resolution to low level routing details
this post is being used more as a way for me to consolodate all of the knowledge i have learned from my networking studies, so if anything is wrong or misinterpiated please let me know


# DNS PROCESS
domain name resolution is a widley used protocol, its main goal is to convert human readable domain names like "facebook.com" into IP addresses
this way instead of trying to remeber "23.24.25.36" we can simply remember "facebook.com"

First we enter Facebook.com into a url bar
Now the dns process takes over, domain name system is a widely used protocol that maps domain names to IPs via A and AAAA records, it also is used for other stuff like
DKIM(stores the public key, used to verify the sender is who they say they are),
DMARC(tells the receiver what to do if there’s something wrong),
SPF(lists the valid servers that can send emails)
CNAMES(used as a sort of forwarding trick)
MX(holds the IP of the mail servers) etc it uses port 53/udp and tcp for zone transfers(zone transfers are when we are transferring all dns configurations over to a new server(zone), dns also uses TCP for query’s if the data is too large for a single UDP packet
it has 5 main steps
First the browser checks the systems local host files, this file stores domain to IP mappings and takes precedence over all others, next it then checks the browsers local cache, this stores recently converted domain names, it then checks the systems local cache, this stores the same as above
If the ip wasn’t found in any of those above it retrieves the dns servers IP from the systems network configuration and sends a dns query to it(I’ll use 8.8.8.8) saying “hey can you please convert Facebook.com for me”
The dns server receives this, this server is also known as a recursive resolver meaning it takes over and handles the rest for us
First the resolver reaches out to a root dns server, there are 13 root dns servers(distributed over 1000s of servers world wide), these servers handle looking the TLD  inside the FQDN the resolver sent and returning the IP of the TLD server that handles the .com TLD, the root server then sends back the IP to the resolver
Now the resolver sends another dns message to the returned IP, this server is known as a TLD server, its holds all the information about the passed TLD, its stores the name servers for whatever registerar the domain was bought from, these name servers are then returned back to the resolver
Finally the resolver chooses one of the nameserver, these servers also known as authoritative dns servers actually store all the dns records like MX, A, AAAA etc, finally the authoritative returns the final IP of the domain we requests
NOTE
All of the mentioned servers also have a cache, they each individually check their cache and when a request is sent to them, I just didn’t specify it for each server for brevity, for big domains like Google their IPs are usually cached and found at step 1 at the resolver
