+++ 
draft = false
date = 2015-05-05
title = "Use IPTables NOTRACK to implement stateless rules and reduce packet loss"
description = ""
summary = "Disabling IPTables connection tracking for DNS can resolve problems due to too many connections in the connection table."
slug = ""
authors = []
tags = ["DNS", "IPTables", "Networking", "Diagnostics"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

I recently struck a performance problem with a high-volume Linux DNS server and found a very satisfying way to overcome it. This post is not about DNS specifically, but useful also to services with a high rate of connections/sessions (UDP or TCP), but it is especially useful for UDP-based traffic, as a stateful firewall doesn't really buy you much with UDP. It is also applicable to services such as HTTP/HTTPS or anything where you have a lot of connections...  

We observed times when DNS would not respond, but retrying very soon after would generally work. For TCP, you may find that you get a a connection timeout (or possibly a connection reset? I haven't checked that recently).  

Observing logs, you might see the following in kernel logs:  

```plain
> kernel: nf_conntrack: table full, dropping packet.
```

You might be inclined to increase net.netfilter.nf_conntrack_max and net.nf_conntrack_max, but a better response might be found by looking at what is actually taking up those entries in your connection tracking table.  
  
We found that the connection tracking was even happening for UDP rules. You could see this with some simple filtering of /proc/net/ip_conntrack looking to see how many entries are there relating to port 53, for example. Here is the basic rules that most Linux people would likely write for iptables.  

```plain
-A INPUT  -p udp --dport 53 -j ACCEPT  
-A INPUT -p tcp --dport 53 -m state --state=NEW -j ACCEPT  
```

## NOTRACK for Stateless Firewall Rules in a Stateful Firewall

Thankfully, I had heard of the NOTRACK rule some time back, but never had a cause to use it, so at least I knew where to begin my research. Red Hat have an article about it at [https://access.redhat.com/solutions/972673](https://access.redhat.com/solutions/972673), though the rules below do not necessarily come from that.  

So we needed to use the 'raw' table to disable stateful inspection for DNS packets; that does mean we need to explictly match all incoming and outgoing packets (which is four UDP flows for a recursive server, plus TCP if you want to do stateless TCP) -- its rather like IPChains way back in the day... and like IPChains, you do lose all the benefits you get from a stateful firewall, and gain all the responsibilities of making sure you explicitly match all traffic flows.  

```plain
*raw  
...  
# Don't do connection tracking for DNS  
-A PREROUTING -p tcp --dport 53 -j NOTRACK  
-A PREROUTING -p udp --dport 53 -j NOTRACK  
-A PREROUTING -p tcp --sport 53 -j NOTRACK  
-A PREROUTING -p udp --sport 53 -j NOTRACK  
-A OUTPUT -p tcp --sport 53 -j NOTRACK  
-A OUTPUT -p udp --sport 53 -j NOTRACK  
-A OUTPUT -p tcp --dport 53 -j NOTRACK  
-A OUTPUT -p udp --dport 53 -j NOTRACK  
...  
COMMIT  

...

*filter
...

# Allow stateless UDP serving
-A INPUT  -p udp --dport 53 -j ACCEPT
-A OUTPUT -p udp --sport 53 -j ACCEPT

# Allow stateless UDP backending
-A OUTPUT -p udp --dport 53 -j ACCEPT
-A INPUT  -p udp --sport 53 -j ACCEPT

# Allow stateless TCP serving
-A INPUT  -p tcp --dport 53 -j ACCEPT
-A OUTPUT -p tcp --sport 53 -j ACCEPT

# Allow stateless TCP backending
-A OUTPUT -p tcp --dport 53 -j ACCEPT
-A INPUT  -p tcp --sport 53 -j ACCEPT

...
COMMIT
```
  

## Beware the moving bottleneck

That worked well... perhaps a little too well. Now the service gets more than it did before, and you need to be prepared for that, as you may find that a new limit (and potential negative behaviour) is reached.  

DNS is particularly prone to having very large spikes of activity due to misconfigured clients. A common problem, particularly from Linux clients, are things like Wireshark, scripts that look up (often using dig/host/nslookup -- use `getent hosts` instead), and not having a local name-service cache.

Assuming you can identify such clients, you could (perhaps in conjunction with fail2ban or similar) have some firewall rules that limit allowable request rates from very specific segments of your network... be very cautious of this though.

These rules would go prior to your filter table rules allowing access (listed earlier).  

```plain
-N DNS_TOO_FREQUENT_BLOCKLIST  
# This chain is where the actual rate limiting is put in place.  
# Note that it is using just the srcip method in its hashing  
-A DNS_TOO_FREQUENT_BLOCKLIST -p udp -m udp --dport 53 -m hashlimit --hashlimit-mode srcip --hashlimit-srcmask 32 --hashlimit-above 10/sec --hashlimit-burst 20 --hashlimit-name dns_too_frequen -m comment --comment "drop_overly_frequent_DNS_requests" -j DROP  
  
# This matches a pair of machines I judged to be innocently bombarding DNS  
# It so happens that they could be nicely summarised with a /31  
# The second line is so we can counters of what made it through  
-A INPUT -s «CLIENT_IP»/31 -j DNS_TOO_FREQUENT_BLOCKLIST  
-A INPUT -s «CLIENT_IP»/31  

#... more rules here as needed
```

## Concluding Remarks

I've been running this configuration now for some time, and am very happy with it. I do intend to implement this technique on other services where I feel it may be needed (Syslog servers certainly)
