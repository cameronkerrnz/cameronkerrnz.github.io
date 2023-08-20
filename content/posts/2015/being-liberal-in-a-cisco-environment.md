+++ 
draft = false
date = 2015-05-05
title = "The importance of being liberal in a Cisco environment"
description = ""
slug = ""
authors = []
tags = ["IPTables","Cisco","Networking","Linux"]
categories = []
externalLink = ""
series = []
+++

> This post was originally published at my old blog Distracted IT on Blogger.com

Okay, so today I grappled with a Cisco sized gorilla and won. <-- me, earlier this year, apparently feeling rather chuffed with myself.  
  
I had recently launched a new service for a client, and a harvester (a particular search engine crawler) on the Internet was experiencing timeouts trying to connect, and so our data was not being harvested. There was no evidence of a problem in the web-server logs because no HTTP request ever had a chance to make it through.

It seems that Cisco products (some unstudied subset, but likely firewalls and NATs) seem to play a bit too fast and loose for Linux’s liking and change packets in ways that makes the Linux firewall (iptables) stateful connection tracking occassionally see such traffic as INVALID. This manifests in in things such as connection timeouts, and as such you won’t notice it in things like webserver logs. In a traffic capture, you may recognise it as a lot of retransmissions.
  
Possibly it is affected by TCP window scaling as well. It is only noticeable to traffic from the internet and as such is most likely noticeable only on production systems accepting connections from the internet.  
  
Others on the internet have certainly experienced the same:  

*   [http://starnixhacks.blogspot.co.nz/2010/11/iptables-state-and-cisco-firewalls.html](http://starnixhacks.blogspot.co.nz/2010/11/iptables-state-and-cisco-firewalls.html)
*   [http://blog.endpoint.com/2009/12/cisco-pix-mangled-packets-and-iptables.html](http://blog.endpoint.com/2009/12/cisco-pix-mangled-packets-and-iptables.html)
*   [http://lists.netfilter.org/pipermail/netfilter/2006-September/066840.html](http://lists.netfilter.org/pipermail/netfilter/2006-September/066840.html)

  
The solution is to enable the `nf_conntrack_tcp_be_liberal` flag:  

> nf_conntrack_tcp_be_liberal - BOOLEAN  
> 0 - disabled (default)  
> not 0 - enabled  
> Be conservative in what you do, be liberal in what you accept from others.  
> If it's non-zero, we mark only out of window RST segments as INVALID.

To make the change (non-persistently)

```bash
echo 1 > /proc/sys/net/netfilter/nf_conntrack_tcp_be_liberal  
```

Post change, I experienced immediate resolution of this issue, much to the relief of stakeholders.  

> THANK YOU! YOUR MAGIC WORKED!!!! Woo, I’m too excited!  
> The test harvest run beautifully with 3,812 records in return.

In my experience of this, devices behind a load-balancer do not seem to be affected, probably because our load-balancer has a habit of stripping window-scaling TCP options I think. Its much more likely to present itself when the server is terminating the TCP connection itself.
