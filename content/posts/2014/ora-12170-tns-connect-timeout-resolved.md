---
title: "ORA-12170: TNS:Connect timeout — resolved"
date: 2014-04-26
draft: false
summary: "ORA-12170"
tags: ["Oracle", "Networking"]
---

> This post was originally posted on my Distracted IT blog, where it was my most popular post by an order of magnitude. Sigh, I don't even really like Oracle.

If you're dealing with Oracle clients, you may be familiar with the error message

> ERROR ORA-12170: TNS:Connect timed out occurred

I was recently asked to investigate such a problem where an application server was having trouble talking to a database server. This issue was blocking progress on a number of projects in our development environment, and our developers' agile post-it note Kanban/Scrum board had a red post-it saying 'Waiting for Cameron', so I thought I should promote it to the front of my rather long list of things I needed to do... it probably also helped that the problem domain was rather interesting to me, and so it ended being a late-night productivity session where I wasn't interrupted and my experimentation wouldn't disrupt others. I think my colleagues are still getting used to seeing email from me at the wee hours of the morning.

This can masquerade as a number of other error strings as well. Here's what you might see in the sqlnet.log file, which you can find in the directory 

_instantclient_home/log/diag/clients/user_USER/host_client_ip_and_port/trace/sqlnet.log_

> Fatal NI connect error 12170.

Some Googling turned up a lot of hits, but none that definitively resolved the problem at hand. The key takeaway from that activity was that it was due to network issues preventing a TCP connection being made to the database listener.

So, how to go about diagnosting this issue? First, let's see what we can learn about this scenario from the reports I received and what we know about interconnectivity in this situation. The first piece of information is that this works fine for developers when connecting from their desktops, which are naturally in a different subnet from the database servers. Secondly, it was only impacting this particular application server, which was a different technology stack and thus a significant difference. Third, it also happened to be in the same subnet as the database server.

Connecting at a TCP level (with `nc -z database-server-ip database-server-port || echo FAIL`) manually, even multiple times very quickly, showed no problems. Perhaps it was due to the technology stack being used. I don't have enough Oracle expertise to use sqlplus to simulate the connection, which would have been useful in exempting or isolating the fault to the technology stack, but it seems that this problem is exhibited at the TCP layer, and is known to be transient, so I can't trust a manual test. It was only the scheduled tasks that were being impacted. Thus it can be considered to be an intermittent issue (the type that, when solved, gives the greatest amount of cerebral pleasure when solved... hence my investigation continuing to 3am).

## Capturing the traffic

Right, time to bring out the "big" guns: tcpdump on the application server to capture traffic known to exhibit the error (as indicated by logs in this case), followed by analysis on my workstation using Wireshark. Remember, Wireshark doesn't belong on servers. Before capturing, let's first consider how and what we want to capture.

The scheduled job that was most affected kicked off every ten minutes, and from the logs, it seems to fail to connect more often than not (about 2/3 probability), so I decided to run the capture for 20 minutes. Run the capture on the application server (at least), as its from there that the trouble is being experienced.

Its a good thing to run tcpdump with a time or space limit, as distractions are common and it can be easy to forget about running capture sessions, and you don't want to create an issues for yourself or others later. If your tcpdump supports it, you can use -G seconds -W rotations to put a time limit in the amount of time captured (note: not the amount of time the capture is running). This is a useful pattern to use when diagnosing intermittent issues. In this case, I would be capturing up to two files, each one covering a period to 20 minutes. If you wanted a space-limit, you could use -C instead of -G. Because I'm running with a time-based rotation, the filename argument needs to contain a formatting argument; in this case I'm asking for ISO8601 format in a form such as 2014-04-26T20:43:34.

I like to avoid running in promiscuous mode if not required, hence the -p option, which makes it running without a capture filter more affordable. However, if not capturing to a file using a capture filter of `not port ssh` is very useful.

```bash
tcpdump -p -i eth0 -G 1200 -W 2 -w /home/me/tnsconnect-%FT%T.pcap
```

Tailing the `sqlnet.log` file, I waited till I saw the error, and stopped `tcpdump`. I then copied the file to my desktop to analyse using Wireshark.

## Analysing the Traffic with Wireshark

It's tempting to simply start scrolling to find the packets of interest, or apply a display filter if you think you know what to look for. However, Wireshark has something else of use, its Expert Info (menu Analyze > Expert Info). This shows alerts that its various base protocol analysers pick up. Looking through this can be useful to give you a first glance.

In this case, it did show up some ARP irregularities, namely that an **address was claimed by multiple MAC addresses**. Considering that one of the addresses was the database server, that was a nice red flag. It certainly helps explain the transient nature of the problem, any why it wasn't affecting connections through the router (it could have done, but routers tend to have a much longer ARP cache lifetime, so it kept the good result for longer). Time to look closer, just to see what was actually happening.

Noting the time shown in the sqlnet.log file, I set Wireshark's time display format to show the time of day, and quickly found the traffic around the time of the connection. I saw it attempting to connect (SYN) but the SYN+ACK never eventuated. Looking more closely, I saw that it was attempting to connect to a different MAC address than the one I expected. Looking a little earlier, to the ARP communication, I saw that the application servers ARP request was being simultaneously answered by two inferfaces (this was mentioned in the Expert Info), and looking at what owned those MAC addresses, they were two interfaces on the same database server.

As would be expected, this database server has a management interface, and another interface over which application (database) traffic would go through... nothing out of the ordinary there. Each interface looked to be configured appropriately from the OS point of view, and was in a different interface.

## Which VLAN is this interface on? CDP to the rescue

To get a response from each interface, each interface must be getting the request... that would mean that those interfaces must be on the same layer-2 segment if the network... the same VLAN.

**Hypothesis: management interface is on the wrong VLAN**. That would make since due to the particular history of that machine.

Hmmm, how to verify what VLAN a port is on? I don't have visibility of that, and as its the middle of the night, no networking staff to ask. but I do have tcpdump and the knowledge that I would expect to see Cisco Discovery Protocol (CDP) frames. So, let's fire up tcpdump with (-v), which is sufficient to parse CDP frames, and wait till we see one. A little googling shows how to capture CDP frames:

```bash
tcpdump -nn -v -i eth0 -s 1500 -c 1 'ether[20:2] == 0x2000'
```

In the output, I was able to see the VLAN that the port was assigned to, and indeed it was incorrect (it was the same as the other interface). I don't expect that this would work for all such discoveries, but its useful... and it also told me which switch port needed to be reconfigured.

So I wrote a change request to the networking team, but naturally that wouldn't get actioned until the daylight kicks in, at the earliest. How can I simulate the fix? Two methods come to mind: the first, and simplest, would be to shut down the offending interfaces but let's say that would cause some other issue, or the conflicting interface is on a machine you don't have control over... then in that case, you can **temporarily bypass ARP for connections from the application server to the database server, by putting in a static ARP entry on the application server** (see the arp command).

That done I was able to continue tailing the logs and found that the problem ceased to occur, and stayed fixed until at least the morning, when the VLAN assignment could be changed.

## Followup activities

One of my favourite things about solving problems like this is asking myself "how can I look for other instances of where this may be causing a problem for me?". In this case, I think it would be useful to deploy arpwatch strategically, looking for things like 'flip flops' in various VLANs, particularly in known troublespots. This is particularly useful if you run arpwatch without email reports, and centralise syslog logging to a remote server, where you could then do some analysis. ARP issues are a common way that devices transiently "fall off the network" or fail to connect.
